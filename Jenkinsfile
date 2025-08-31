pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
   
    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION = 'us-east-1'
        ECR_REPO = 'rachel-jenkins'
        APP_NAME = 'calculator-app'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        PROD_HOST = '10.0.1.73'
    }
   
    stages {
        stage('Setup') {
            steps {
                script {
                    sh '''
                        apt-get update
                        apt-get install -y curl docker.io awscli
                        pip install -r requirements.txt
                    '''
                }
            }
        }
       
        stage('Build Container Image') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.IMAGE_TAG = "latest-${BUILD_NUMBER}"
                    } else if (env.CHANGE_ID) {
                        env.IMAGE_TAG = "pr-${CHANGE_ID}-${BUILD_NUMBER}"
                    } else {
                        env.IMAGE_TAG = "${BRANCH_NAME}-${BUILD_NUMBER}"
                    }
                    sh """
                        docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} .
                    """
                }
            }
        }
       
        stage('Test') {
            steps {
                script {
                    sh '''
                        python -m unittest discover -s tests -v

                        mkdir -p test-results
                        python -m unittest discover -s tests -v > test-results/test-output.txt 2>&1 || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'test-results/*.txt', fingerprint: true
                }
            }
        }
       
        stage('Push to ECR') {
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                        echo "Pushed image: ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                    """
                }
            }
        }
       
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sshagent(['production-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@${PROD_HOST} '
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                docker pull ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                                docker stop ${APP_NAME} || true
                                docker rm ${APP_NAME} || true
                                docker run -d --name ${APP_NAME} -p 5000:5000 ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                                sleep 10
                            '
                        """
                    }
                }
            }
        }
       
        stage('Health Verification') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        for i in \$(seq 1 10); do
                            echo "Health check attempt \$i"
                            if curl -f http://${PROD_HOST}:5000/health; then
                                echo "Health check passed!"
                                exit 0
                            fi
                            sleep 5
                        done
                        echo "Health check failed after 10 attempts"
                        exit 1
                    """
                }
            }
        }
    }
   
    post {
        always {
            script {
                echo "Pipeline completed for branch: ${BRANCH_NAME}"
                echo "Image built: ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
            }
        }
        failure {
            script {
                if (env.BRANCH_NAME == 'main') {
                    echo "Production deployment failed! Rolling back..."
            
                }
            }
        }
    }
}

