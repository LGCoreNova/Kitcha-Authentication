pipeline {
    agent any
    
    environment {
        SERVICE_NAME = "auth"
        DOCKER_HUB_CREDS = credentials('docker-hub-credentials')
        DOCKER_IMAGE = "wdd1016/auth"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh './gradlew clean build -x test'
            }
        }
        
        // stage('Test') {
        //     steps {
        //         sh './gradlew test'
        //     }
        //     post {
        //         always {
        //             junit '**/build/test-results/test/*.xml'
        //         }
        //     }
        // }
        
        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} -t ${DOCKER_IMAGE}:latest ."
            }
        }
        
        stage('Docker Push') {
            steps {
                sh "echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin"
                sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                sh "docker push ${DOCKER_IMAGE}:latest"
            }
        }
        
        stage('Deploy') {
            steps {
                // 배포 스크립트 실행
                sshagent(['ec2-deploy-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@your-target-server '
                        cd /path/to/kitcha &&
                        docker-compose pull ${SERVICE_NAME} &&
                        docker-compose up -d --no-deps ${SERVICE_NAME}
                        '
                    """
                }
            }
        }
    }
    
    post {
        always {
            // 작업 공간 정리
            cleanWs()
        }
        success {
            // 슬랙 등 알림 전송 (선택사항)
            echo "Build succeeded!"
        }
        failure {
            echo "Build failed!"
        }
    }
}