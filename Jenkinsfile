pipeline {
    agent any

    environment {
        // ====> Replace with your AWS region, e.g., 'us-east-1'
        AWS_REGION = 'your-aws-region'

        // ====> Replace with your own ECR repository URIs
        FRONTEND_REPO = 'your-frontend-ecr-repo-uri'
        BACKEND_REPO  = 'your-backend-ecr-repo-uri'
    }

    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker images') {
            steps {
                script {
                    sh 'docker build -t frontend:latest ./frontend'
                    sh 'docker build -t backend:latest ./backend'
                }
            }
        }

        stage('Authenticate to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'your-aws-credentials-id']]) {
                    script {
                        sh '''
                            aws --version
                            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $FRONTEND_REPO
                            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $BACKEND_REPO
                        '''
                    }
                }
            }
        }

        stage('Tag and Push images to ECR') {
            steps {
                script {
                    sh '''
                        docker tag frontend:latest $FRONTEND_REPO:latest
                        docker tag backend:latest $BACKEND_REPO:latest

                        docker push $FRONTEND_REPO:latest
                        docker push $BACKEND_REPO:latest
                    '''
                }
            }
        }

        stage('Update ECS services') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'your-aws-credentials-id']]) {
                    script {
                        sh '''
                            aws ecs update-service --cluster your-ecs-cluster-name --service your-frontend-service-name --force-new-deployment --region $AWS_REGION
                            aws ecs update-service --cluster your-ecs-cluster-name --service your-backend-service-name --force-new-deployment --region $AWS_REGION
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
