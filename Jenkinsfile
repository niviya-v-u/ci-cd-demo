pipeline {
    agent any

    environment {
        AWS_REGION      = 'ap-south-1'
        AWS_ACCOUNT_ID  = '196177110673'
        ECR_REPO_NAME   = 'ci-cd-demo'
        IMAGE_TAG       = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning repository..."
                checkout scm
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo "Building & pushing Docker image (linux/amd64)..."
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-ecr-credentials'
                ]]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        docker buildx create --use || true
                        
                        docker buildx build \
                            --platform linux/amd64 \
                            -t ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG} \
                            --push .
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "Deploying to EC2..."
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@52.66.240.113 '
                            aws ecr get-login-password --region ${AWS_REGION} \
                                | sudo docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com &&
                            
                            sudo docker pull ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG} &&

                            sudo docker stop myapp || true &&
                            sudo docker rm myapp || true &&

                            sudo docker run -d -p 80:8080 --name myapp \
                                ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "SUCCESS! Build ${IMAGE_TAG} deployed."
        }
        failure {
            echo "FAILURE! Check Jenkins logs."
        }
    }
}
