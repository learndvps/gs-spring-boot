pipeline {
    agent any
    
    tools {
        maven 'Maven-3.8.6'
        jdk 'Java-17'
    }
    
    environment {
        DOCKER_IMAGE = 'spring-boot-app'
        QA_SERVER = '<QA87-PRIVATE-IP>'
        DEPLOY_PATH = '/opt/deployment'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/gs-spring-boot.git'
            }
        }
        
        stage('Build') {
            steps {
                dir('complete') {
                    sh 'mvn clean compile'
                }
            }
        }
        
        stage('Test') {
            steps {
                dir('complete') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    dir('complete') {
                        publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                dir('complete') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                dir('complete') {
                    sh 'docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
                    sh 'docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest'
                }
            }
        }
        
        stage('Deploy to QA87') {
            steps {
                sshagent(['qa87-ssh']) {
                    sh """
                        # Stop and remove existing container
                        ssh -o StrictHostKeyChecking=no ec2-user@${QA_SERVER} '
                            docker stop spring-boot-app || true
                            docker rm spring-boot-app || true
                        '
                        
                        # Save Docker image to tar file
                        docker save ${DOCKER_IMAGE}:latest > spring-boot-app.tar
                        
                        # Copy image to QA87
                        scp -o StrictHostKeyChecking=no spring-boot-app.tar ec2-user@${QA_SERVER}:${DEPLOY_PATH}/
                        
                        # Load and run image on QA87
                        ssh -o StrictHostKeyChecking=no ec2-user@${QA_SERVER} '
                            cd ${DEPLOY_PATH}
                            docker load < spring-boot-app.tar
                            docker run -d --name spring-boot-app -p 8080:8080 ${DOCKER_IMAGE}:latest
                            rm spring-boot-app.tar
                        '
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sleep(30) // Wait for application to start
                    sh """
                        curl -f http://${QA_SERVER}:8080/actuator/health || exit 1
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}