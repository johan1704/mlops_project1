pipeline {
    agent any
    
    environment {
        VENV_DIR = 'venv'
        AWS_REGION = 'eu-west-1'
        AWS_ACCOUNT_ID = '297984596884'
        ECR_REPO_NAME = 'mlops-project'
        APP_RUNNER_SERVICE = 'mlops-app-runner-service'
    }

    stages {
        stage('Cloning Github repo to Jenkins') {
            steps {
                script {
                    echo 'Cloning github repo to Jenkins..'
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/johan1704/mlops_project1.git']])
                }
            }
        }

        stage('Setting up Docker permissions') {
            steps {
                script {
                    echo 'Setting up Docker permissions..'
                    sh '''
                    sudo usermod -a -G docker jenkins || true
                    sudo chmod 666 /var/run/docker.sock || true
                    '''
                }
            }
        }

        stage('Setting up our venv environment and dependencies') {
            steps {
                script {
                    echo 'Setting up our venv environment and dependencies..'
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
                    $class: 'UsernamePasswordMultiBinding',
                    credentialsId: 'aws-key',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo 'Building and Pushing Docker Image to Amazon ECR.............'
                        sh '''
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        
                        PASSWORD=$(aws ecr get-login-password --region ${AWS_REGION})
                        echo $PASSWORD | sudo docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        if ! aws ecr describe-repositories --repository-names ${ECR_REPO_NAME} --region ${AWS_REGION} 2>/dev/null; then
                            echo "Creating ECR repository: ${ECR_REPO_NAME}"
                            aws ecr create-repository --repository-name ${ECR_REPO_NAME} --region ${AWS_REGION}
                            sleep 5
                        else
                            echo "ECR repository ${ECR_REPO_NAME} already exists"
                        fi

                        sudo docker build -t ${ECR_REPO_NAME}:latest .
                        sudo docker tag ${ECR_REPO_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                        sudo docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy to AWS App Runner') {
            steps {
                withCredentials([[
                    $class: 'UsernamePasswordMultiBinding',
                    credentialsId: 'aws-key',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo 'Deploy to AWS App Runner.............'
                        sh '''
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        IMAGE_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest"

                        SERVICE_ARN=$(aws apprunner list-services --query "ServiceSummaryList[?ServiceName=='${APP_RUNNER_SERVICE}'].ServiceArn" --output text)

                        if [ -z "$SERVICE_ARN" ]; then
                            echo "Creating new App Runner service: ${APP_RUNNER_SERVICE}"
                            aws apprunner create-service \
                                --service-name ${APP_RUNNER_SERVICE} \
                                --source-configuration "ImageRepository={ImageIdentifier=${IMAGE_URI},ImageRepositoryType=ECR}" \
                                --instance-configuration "Cpu=1024,Memory=2048" \
                                --region ${AWS_REGION}
                        else
                            echo "Updating existing App Runner service: ${APP_RUNNER_SERVICE}"
                            aws apprunner update-service \
                                --service-arn ${SERVICE_ARN} \
                                --source-configuration "ImageRepository={ImageIdentifier=${IMAGE_URI},ImageRepositoryType=ECR}" \
                                --region ${AWS_REGION}
                        fi
                        '''
                    }
                }
            }
        }
    }
}