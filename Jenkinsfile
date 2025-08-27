pipeline{
    agent any
    
    environment {
        VENV_DIR = 'venv'
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '297984596884'
        ECR_REPO_NAME = 'mlops-project'
    }

    stages{
        stage('cloning github repo to jenkins'){
            steps{
                script{
                    echo 'cloning github repo ..'
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/johan1704/mlops_project1.git']])
                }
            }
        }

        stage('setting up our venv environment and dependencies'){
            steps{
                script{
                    echo 'setting up our venv environment and dependencies ..'
                    sh '''
                    python -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -e .
                    '''
                }
            }
        }
        stage('Building and Pushing Docker Image to Amazon ECR') {
            steps {
                withCredentials([[
                    credentialsId: 'aws-key',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo 'Building and Pushing Docker Image to Amazon ECR.............'
                        sh '''
                        # Configure AWS CLI
                        aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                        aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                        aws configure set region ${AWS_REGION}

                        # Login to ECR
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        # Create ECR repository if it doesn't exist
                        aws ecr describe-repositories --repository-names ${ECR_REPO_NAME} || aws ecr create-repository --repository-name ${ECR_REPO_NAME}

                        # Build and tag Docker image
                        docker build -t ${ECR_REPO_NAME}:latest .
                        docker tag ${ECR_REPO_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest

                        # Push to ECR
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy to AWS ECS/Fargate') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-key',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo 'Deploy to AWS ECS/Fargate.............'
                        sh '''
                        # Configure AWS CLI
                        aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                        aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                        aws configure set region ${AWS_REGION}

                        # Update ECS service with new task definition
                        # This assumes you already have an ECS cluster and service set up
                        aws ecs update-service \
                            --cluster mlops-cluster \
                            --service ml-project-task-service-f57ehlbn \
                            --force-new-deployment
                        '''
                    }
                }
            }
        }

        }
    }

    post {
        always {
            script {
                echo 'Cleaning up...'
                sh 'docker system prune -f'
            }
        }
    }
