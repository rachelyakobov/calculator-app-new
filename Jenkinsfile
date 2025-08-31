pipeline {
    agent {
        docker {
            image 'docker:stable'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION = 'us-east-1'
        ECR_REPO = 'rachel-jenkins'
        APP_NAME = 'calculator-app'
    }
    stages {
        stage('Login to AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }
        stage('Build Image') {
            steps {
                script {
                    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
                }
                sh "docker build -t $IMAGE_URI ."
            }
        }
        stage('Run Tests') {
            steps {
                sh '''
                    python -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements.txt
                    python -m unittest discover -s tests -v
                '''
            }
        }
        stage('Push Image to ECR') {
            steps {
                sh "docker push $IMAGE_URI"
            }
        }
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    PRODUCTION_HOST = "production-ec2-host"
                    PRODUCTION_IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:latest"
                }
                sh """
                    ssh -o StrictHostKeyChecking=no ec2-user@${PRODUCTION_HOST} '
                        docker pull ${PRODUCTION_IMAGE_URI} &&
                        docker stop ${APP_NAME} || true &&
                        docker rm ${APP_NAME} || true &&
                        docker run -d --name ${APP_NAME} -p 5000:5000 ${PRODUCTION_IMAGE_URI}
                    '
                """
            }
        }
        stage('Health Check') {
            when {
                branch 'main'
            }
            steps {
                script {
                    PRODUCTION_HOST = "production-ec2-host"
                }
                retry(5) {
                    sh """
                        ssh ec2-user@${PRODUCTION_HOST} '
                            curl -fsS http://localhost:5000/health
                        '
                    """
                }
            }
        }
    }
}

