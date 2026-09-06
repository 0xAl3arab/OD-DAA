pipeline {
    agent { label 'built-in' }

    tools {
        maven 'Maven-3.9.16'
    }

    environment {
        DOCKER_IMAGE           = 'main-app'                     // <-- your image name
        NEXUS_DOCKER_REGISTRY  = 'host.docker.internal:8082'
        IMAGE_TAG              = "${env.GIT_COMMIT.take(7)}"
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
                script {
                    env.APP_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarServer') {
                    sh '''
                        mvn verify \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.projectKey=Main-App \
                          -Dsonar.projectName=Main-App
                    '''
                }
            }
        }

        stage('Install Semgrep') {
            steps {
                sh 'pip3 install semgrep --break-system-packages --user'
            }
        }

        stage('Semgrep SAST') {
            steps {
                withCredentials([string(credentialsId: 'semgrep-token', variable: 'SEMGREP_APP_TOKEN')]) {
                    sh '''
                        export PATH="$HOME/.local/bin:$PATH"
                        semgrep --config=auto --json --output=semgrep-results.json . || true
                    '''
                }
            }
        }

        stage('Snyk SCA Scan') {
            steps {
                withCredentials([string(credentialsId: 'Snyk', variable: 'SNYK_TOKEN')]) {
                    sh '''
                        SNYK_BIN=/var/jenkins_home/tools/io.snyk.jenkins.tools.SnykInstallation/Snyk/snyk-linux
                        $SNYK_BIN auth "$SNYK_TOKEN" || true
                        $SNYK_BIN test --org=0394a8ef-9320-4dce-8cf8-ce5a1e7a4694 --json --severity-threshold=low > snyk-report.json 2> snyk-debug.log || true
                    '''
                }
            }
        }

        // ---- Everything below only runs after merge to main, not on PRs ----

        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                sh "docker build -t ${NEXUS_DOCKER_REGISTRY}/${DOCKER_IMAGE}:${APP_VERSION}-${IMAGE_TAG} ."
                sh "docker tag ${NEXUS_DOCKER_REGISTRY}/${DOCKER_IMAGE}:${APP_VERSION}-${IMAGE_TAG} ${NEXUS_DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push Docker Image to Nexus') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        mkdir -p ~/.docker
                        AUTH=$(echo -n "$NEXUS_USER:$NEXUS_PASS" | base64 -w 0)
                        cat > ~/.docker/config.json << EOF
{
  "auths": {
    "${NEXUS_DOCKER_REGISTRY}": {
      "auth": "$AUTH"
    }
  }
}
EOF
                        docker push ${NEXUS_DOCKER_REGISTRY}/${DOCKER_IMAGE}:${APP_VERSION}-${IMAGE_TAG}
                        docker push ${NEXUS_DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(artifacts: 'semgrep-results.json, snyk-report.json, snyk-debug.log', allowEmptyArchive: true)
        }
    }
}