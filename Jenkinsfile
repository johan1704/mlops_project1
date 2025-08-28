pipeline {
    agent any
    
    environment {
        VENV_DIR = 'venv'
        AWS_REGION = 'eu-north-1'
        AWS_ACCOUNT_ID = '297984596884'
        ECR_REPO_NAME = 'mlops-project'
        ECS_CLUSTER = 'arn:aws:ecs:eu-north-1:297984596884:cluster/mlops-cluster'
        ECS_SERVICE = 'ml-project-task-service-f57ehlbn'
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
                    # Ajouter l'utilisateur jenkins au groupe docker
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

        stage('Create Missing IAM Role') {
            steps {
                withCredentials([[
                    $class: 'UsernamePasswordMultiBinding',
                    credentialsId: 'aws-key',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo 'Creating missing IAM role: ecsTaskExecutionRole.............'
                        sh '''
                        # Configure AWS environment variables
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        # Créer le rôle manquant
                        aws iam create-role \
                          --role-name ecsTaskExecutionRole \
                          --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

                        # Attacher la politique nécessaire
                        aws iam attach-role-policy \
                          --role-name ecsTaskExecutionRole \
                          --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

                        echo "Waiting for IAM role propagation..."
                        sleep 10
                        '''
                    }
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
                        # Configure AWS environment variables
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        
                        # Login to ECR
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        # Créer le repository ECR s'il n'existe pas
                        if ! aws ecr describe-repositories --repository-names ${ECR_REPO_NAME} --region ${AWS_REGION} 2>/dev/null; then
                            echo "Creating ECR repository: ${ECR_REPO_NAME}"
                            aws ecr create-repository --repository-name ${ECR_REPO_NAME} --region ${AWS_REGION}
                            sleep 5 # Attendre un peu que le repository soit complètement créé
                        else
                            echo "ECR repository ${ECR_REPO_NAME} already exists"
                        fi

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
                    $class: 'UsernamePasswordMultiBinding',
                    credentialsId: 'aws-key',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo 'Deploy to AWS ECS/Fargate.............'
                        sh '''
                        # Configure AWS environment variables
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        # Update ECS service with new task definition
                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER} \
                            --service ${ECS_SERVICE} \
                            --force-new-deployment
                        '''
                    }
                }
            }
        }
    }
}