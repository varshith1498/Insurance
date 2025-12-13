pipeline {
    agent any

    environment {
        APP_SERVER_IP = '18.217.36.36'
        APP_USER      = 'ubuntu'
        DEPLOY_DIR    = '/home/ubuntu/app'
        DEPLOY_PATH   = '/home/ubuntu/app/app.jar'

        TRIVY_CACHE_DIR = "${WORKSPACE}/.trivycache"
        TRIVY_DISABLE_VEX_NOTICE = "true"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps {
                sh '''
                  mvn -version
                  mvn clean install
                '''
            }
        }

        stage('Test') {
            steps { sh 'mvn test' }
        }

        stage('Security Scan - Trivy (Report Only)') {
            steps {
                sh '''
                  echo "Running Trivy scan (REPORT ONLY - will not fail pipeline)"
                  mkdir -p reports
                  mkdir -p "$TRIVY_CACHE_DIR"

                  trivy fs \
                    --cache-dir "$TRIVY_CACHE_DIR" \
                    --severity HIGH,CRITICAL \
                    --format table \
                    -o reports/trivy-report.txt \
                    . || true

                  echo "----- Trivy Report Summary -----"
                  tail -n 60 reports/trivy-report.txt || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/trivy-report.txt', fingerprint: true
                }
            }
        }

        stage('Package') {
            steps { sh 'mvn package' }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Deploy to Application Server (Report Only)') {
            when { branch 'main' }
            steps {
                // catchError makes this stage NOT fail the pipeline
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sshagent(credentials: ['appcreds']) {
                        sh '''
                          set +e

                          echo "Selecting artifact..."
                          JAR_FILE=$(ls -1 target/*.jar | head -n 1)
                          echo "Artifact: $JAR_FILE"

                          echo "Preparing app folder on server..."
                          ssh -o StrictHostKeyChecking=no ${APP_USER}@${APP_SERVER_IP} "mkdir -p ${DEPLOY_DIR}"
                          echo "mkdir status=$?"

                          echo "Copying artifact to app server..."
                          scp -o StrictHostKeyChecking=no "$JAR_FILE" ${APP_USER}@${APP_SERVER_IP}:${DEPLOY_PATH}
                          echo "scp status=$?"

                          echo "Restarting application on app server..."
                          ssh -o StrictHostKeyChecking=no ${APP_USER}@${APP_SERVER_IP} "
                            set +e
                            echo '--- SERVER DEBUG ---'
                            whoami
                            hostname
                            pwd
                            echo 'Java version:'; java -version
                            echo 'Killing old process (if any)...'
                            pkill -f '${DEPLOY_PATH}' || true

                            echo 'Starting app...'
                            nohup java -jar '${DEPLOY_PATH}' > '${DEPLOY_DIR}/app.log' 2>&1 &
                            sleep 2

                            echo 'Process check:'
                            ps -ef | grep '${DEPLOY_PATH}' | grep -v grep || true

                            echo 'Last 80 lines of app.log:'
                            tail -n 80 '${DEPLOY_DIR}/app.log' || true

                            echo '--- END SERVER DEBUG ---'
                            exit 0
                          "
                          echo "ssh restart status=$?"

                          echo "Deploy finished (report-only)."
                          exit 0
                        '''
                    }
                }
            }
        }
    }

    post {
        always { echo "Pipeline finished." }
    }
}
