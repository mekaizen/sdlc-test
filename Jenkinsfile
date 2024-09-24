pipeline {
    agent any
    environment {
        SONAR_TOKEN = credentials('SONAR_TOKEN1') // Assuming you've set up a secret for SonarQube
    }
    stages {
        stage('Checkout') {
            steps {
//                 git 'https://github.com/mekaizen/sdlc-test.git'
                            git branch: 'main', url: 'https://github.com/mekaizen/sdlc-test.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('SAST Scan') {
            steps {
                sh '''
                     mvn -Dmaven.test.failure.ignore verify sonar:sonar \
                     -Dsonar.login=$SONAR_TOKEN1 \
                     -Dsonar.projectKey=sdlc-test \
                     -Dsonar.host.url=http://localhost:9000/
                '''
            }
        }
        stage('Start Application') {
            steps {
                script {
                    // Start the application for testing locally
                    sh '''
            nohup java -jar target/sdlc-test-0.0.1-SNAPSHOT.jar > app.log 2>&1 &                    '''
                    // Wait a few seconds for the app to start
                    sleep 5
                }
            }
        }
        stage('ZAP Baseline Scan') {
            steps {
                script {
                   // Pull the ZAP Docker image and run the scan against the local app
                              sh '''
                                              # Pull the ZAP Docker image
                                              docker pull zaproxy/zap-stable

                                              # Create a temporary directory for ZAP
                                              mkdir -p ${WORKSPACE}/zap_temp
                                              chown 130:139 ${WORKSPACE}/zap_temp  # Ensure proper permissions

                                              # Remove existing report file if it exists
                                              if [ -f ${WORKSPACE}/zap_report.html ]; then
                                                  rm ${WORKSPACE}/zap_report.html
                                              fi

                                              # Run the ZAP baseline scan
                                              docker run --network="host" \
                                                  -v ${WORKSPACE}:/zap/wrk \
                                                  -v ${WORKSPACE}/zap_temp:/home/zap \
                                                  --user=130:139 \
                                                  zaproxy/zap-stable zap-baseline.py -t http://localhost:8080 -r /zap/wrk/zap_report.html || {
                                                      echo "ZAP Baseline Scan failed"
                                                      exit 1
                                                  }
                               '''
                }
            }
        }
        stage('Publish ZAP Report') {
            steps {
                script {
                    // Publish the ZAP scan report in Jenkins
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: '.', // Directory where the ZAP report is saved
                        reportFiles: 'zap_report.html',
                        reportName: 'ZAP Security Report'
                    ])
                }
            }
        }

          stage('Publish') {
                    steps {
                        sh 'mvn deploy'
                    }
                }

                stage('Deploy to Kubernetes') {
                    steps {
                        sh 'kubectl apply -f k8s/deployment.yaml'
                    }
                }

    }
    post {
        success {
            echo 'Build, test, and ZAP scan passed'
        }
        failure {
            echo 'Build failed'
        }
        always {
            // Clean up after the build
            script {
                sh '''
                docker container prune -f
                kill $(ps aux | grep 'sdlc-test.jar' | awk '{print $2}')
                '''
            }
        }
    }
}
