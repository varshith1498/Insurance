pipeline {
    agent any

    environment {
        APP_SERVER_IP = '18.217.36.36'             // ✅ removed trailing space
        APP_USER      = 'ubuntu'
        DEPLOY_PATH   = '/home/ubuntu/app/app.jar'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -version'
                sh 'mvn clean install'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Security Scan - Trivy') {
            steps {
                sh '''
                  echo "Running Trivy filesystem scan"
                  trivy fs --severity HIGH,CRITICAL --exit-code 1 .
                '''
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy to Application Server') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh '''
                      set -e

                      echo "Selecting artifact..."
                      JAR_FILE=$(ls -1 target/*.jar | head -n 1)
                      echo "Artifact: $JAR_FILE"

                      echo "Copying artifact to app server..."
                      scp -o StrictHostKeyChecking=no "$JAR_FILE" \
                        ${APP_USER}@${APP_SERVER_IP}:${DEPLOY_PATH}

                      echo "Restarting application on app server..."
                      ssh -o StrictHostKeyChecking=no ${APP_USER}@${APP_SERVER_IP} "
                        mkdir -p /home/ubuntu/app
                        pkill -f 'java -jar' || true
                        nohup java -jar ${DEPLOY_PATH} > /home/ubuntu/app/app.log 2>&1 &
                        sleep 2
                        echo 'App started. Tail logs with: tail -n 50 /home/ubuntu/app/app.log'
                      "
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished."
        }
    }
}
