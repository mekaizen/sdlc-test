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




stage('ZAP Automation Framework Scan') {
    steps {
        script {
            sh '''
                docker pull zaproxy/zap-stable

                # Ensure proper permissions for the zap_temp directory
                mkdir -p ${WORKSPACE}/zap_temp
                chmod 777 ${WORKSPACE}/zap_temp

                # Ensure the zap_config directory is properly set up
                mkdir -p ${WORKSPACE}/zap_config
                chmod 777 ${WORKSPACE}/zap_config

                # Move zap.yaml to zap_config if needed
                if [ -f ${WORKSPACE}/zap_temp/zap.yaml ]; then
                    mv ${WORKSPACE}/zap_temp/zap.yaml ${WORKSPACE}/zap_config/
                fi

                # Run the ZAP Automation Framework using -autorun
                docker run --network="host" \
                    -v ${WORKSPACE}:/zap/wrk \
                    -v ${WORKSPACE}/zap_temp:/home/zap \
                    -v ${WORKSPACE}/zap_config:/zap/config \
                    zaproxy/zap-stable zap.sh -cmd -autorun /zap/config/zap.yaml || {
                        echo "ZAP Automation Framework Scan failed"
                        exit 1
                    }

                echo "Contents of zap_out.json:"
                cat ${WORKSPACE}/zap_temp/zap_out.json || echo "zap_out.json is empty"
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
