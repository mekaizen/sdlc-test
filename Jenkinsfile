pipeline {
    agent any
    environment {
        SONAR_TOKEN = credentials('SONAR_TOKEN1')
    }
    stages {
        stage('Checkout') {
            steps {
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
                     -Dsonar.login=$SONAR_TOKEN \
                     -Dsonar.projectKey=sdlc-test \
                     -Dsonar.host.url=http://localhost:9000/
                '''
            }
        }
        stage('Start Application') {
            steps {
                script {
                    // Start the application for testing locally on port 8081
                sh '''
                                nohup java -jar target/sdlc-test-0.0.1-SNAPSHOT.jar --server.port=8084 > app.log 2>&1 &
                            '''
                            // Increase sleep time to ensure application fully starts
                            sleep 10
                }
            }
        }
        stage('Verify Application Running') {
            steps {
                script {
                    // Check if the application is running and accessible on port 8081
                    sh '''
                        curl -I http://localhost:8084 || {
                            echo "Application is not running!"
                            exit 1
                        }
                    '''
                }
            }
        }
        stage('Check Application Logs') {
            steps {
                script {
                    // Output the application logs
                    sh 'cat app.log'
                }
            }
        }

        stage('ZAP Baseline Scan') {
            steps {
                script {
                   // Pull the ZAP Docker image and run the scan against the local app on port 8081
                    sh '''
                        docker pull zaproxy/zap-stable

                        mkdir -p ${WORKSPACE}/zap_temp
                        chown 130:139 ${WORKSPACE}/zap_temp  # Ensure proper permissions

                        if [ -f ${WORKSPACE}/zap_report.html ]; then
                            rm ${WORKSPACE}/zap_report.html
                        fi

                        echo "Contents of workspace:"
                        ls -la ${WORKSPACE}

                        echo "Contents of zap_temp:"
                        ls -la ${WORKSPACE}/zap_temp

                        docker run --network="host" \
                            -v ${WORKSPACE}:/zap/wrk \
                            -v ${WORKSPACE}/zap_temp:/home/zap \
                            --user=130:139 \
                            zaproxy/zap-stable zap-baseline.py -t http://localhost:8084 || {
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
            script {
                sh '''
                docker container prune -f
                kill $(ps aux | grep 'sdlc-test.jar' | awk '{print $2}')
                '''
            }
        }
    }
}
