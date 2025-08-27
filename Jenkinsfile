pipeline {
    agent any
    environment {
        VENV_DIR = 'venv'
        AWS_DEFAULT_REGION = 'us-east-1'  // Changez selon votre région préférée
        AWS_ACCOUNT_ID = '297984596884' // Vous devrez remplir votre Account ID AWS
        ECR_REPOSITORY = 'mlops-project'
        ECS_CLUSTER = 'ml-project-cluster'
        ECS_SERVICE = 'ml-project-service'
        ECS_TASK_DEFINITION = 'ml-project-task'
    }
    stages {
        stage('Cloning Github repo to Jenkins') {
            steps {
                script {
                    echo 'Cloning Github repo to Jenkins............'
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/data-guru0/MLOPS-COURSE-PROJECT-1.git']])
                }
            }
        }
        
        stage('Setting up Virtual Environment and Installing dependencies') {
            steps {
                script {
                    echo 'Setting up Virtual Environment and Installing dependencies............'
                    sh '''
                    python -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -e .
                    '''
                }
            }
        }
        
        stage('Get AWS Account ID') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                    script {
                        echo 'Verifying AWS Account ID............'
                        def accountId = sh(
                            script: 'aws sts get-caller-identity --query Account --output text',
                            returnStdout: true
                        ).trim()
                        echo "AWS Account ID confirmed: ${accountId}"
                        // Verify it matches the hardcoded one
                        if (accountId != env.AWS_ACCOUNT_ID) {
                            echo "WARNING: Account ID mismatch. Using detected ID: ${accountId}"
                            env.AWS_ACCOUNT_ID = accountId
                        }
                    }
                }
            }
        }
        
        stage('Building and Pushing Docker Image to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                    script {
                        echo 'Building and Pushing Docker Image to ECR.............'
                        sh '''
                        # Login to ECR
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                        
                        # Build Docker image
                        docker build -t ${ECR_REPOSITORY}:latest .
                        
                        # Tag image for ECR
                        docker tag ${ECR_REPOSITORY}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest
                        
                        # Push to ECR
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest
                        '''
                    }
                }
            }
        }
        
        stage('Setup AWS Network Configuration') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                    script {
                        echo 'Setting up AWS Network Configuration.............'
                        sh '''
                        # Get default VPC
                        VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)
                        echo "Default VPC ID: $VPC_ID"
                        
                        # Get public subnets from default VPC
                        SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=default-for-az,Values=true" --query 'Subnets[*].SubnetId' --output text)
                        echo "Available Subnet IDs: $SUBNET_IDS"
                        
                        # Convert space-separated list to comma-separated for ECS
                        FORMATTED_SUBNETS=$(echo $SUBNET_IDS | sed 's/ /,/g')
                        echo "Formatted Subnets: $FORMATTED_SUBNETS"
                        
                        # Check if security group exists, create if not
                        SG_EXISTS=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=ml-project-ecs-sg" --query 'SecurityGroups[0].GroupId' --output text 2>/dev/null || echo "None")
                        
                        if [ "$SG_EXISTS" = "None" ] || [ "$SG_EXISTS" = "" ]; then
                            echo "Creating security group..."
                            SECURITY_GROUP_ID=$(aws ec2 create-security-group \
                                --group-name ml-project-ecs-sg \
                                --description "Security group for ML Project ECS" \
                                --vpc-id $VPC_ID \
                                --query 'GroupId' --output text)
                            echo "Created Security Group: $SECURITY_GROUP_ID"
                            
                            # Allow HTTP traffic on port 8080
                            aws ec2 authorize-security-group-ingress \
                                --group-id $SECURITY_GROUP_ID \
                                --protocol tcp \
                                --port 8080 \
                                --cidr 0.0.0.0/0
                            echo "Added ingress rule for port 8080"
                        else
                            SECURITY_GROUP_ID=$SG_EXISTS
                            echo "Using existing Security Group: $SECURITY_GROUP_ID"
                        fi
                        
                        # Create CloudWatch log group if it doesn't exist
                        aws logs create-log-group --log-group-name /ecs/ml-project 2>/dev/null || echo "Log group already exists"
                        
                        # Save network configuration for next stage
                        echo "$FORMATTED_SUBNETS" > subnets.txt
                        echo "$SECURITY_GROUP_ID" > security_group.txt
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to AWS ECS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                    script {
                        echo 'Deploying to AWS ECS.............'
                        sh '''
                        # Read network configuration from previous stage
                        SUBNETS=$(cat subnets.txt)
                        SECURITY_GROUP=$(cat security_group.txt)
                        echo "Using Subnets: $SUBNETS"
                        echo "Using Security Group: $SECURITY_GROUP"
                        
                        # Create or update ECS task definition
                        cat > task-definition.json << EOF
{
    "family": "${ECS_TASK_DEFINITION}",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "ml-project-container",
            "image": "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest",
            "portMappings": [
                {
                    "containerPort": 8080,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/ml-project",
                    "awslogs-region": "${AWS_DEFAULT_REGION}",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
EOF

                        # Register the task definition
                        aws ecs register-task-definition --cli-input-json file://task-definition.json
                        
                        # Check if cluster exists, create if not
                        if ! aws ecs describe-clusters --clusters ${ECS_CLUSTER} --query 'clusters[0].clusterName' --output text 2>/dev/null; then
                            echo "Creating ECS cluster..."
                            aws ecs create-cluster --cluster-name ${ECS_CLUSTER}
                        fi
                        
                        # Check if service exists
                        if aws ecs describe-services --cluster ${ECS_CLUSTER} --services ${ECS_SERVICE} --query 'services[0].serviceName' --output text 2>/dev/null | grep -q ${ECS_SERVICE}; then
                            echo "Updating existing service..."
                            aws ecs update-service \
                                --cluster ${ECS_CLUSTER} \
                                --service ${ECS_SERVICE} \
                                --task-definition ${ECS_TASK_DEFINITION}
                        else
                            echo "Creating new service..."
                            # Use dynamically retrieved subnets and security group
                            aws ecs create-service \
                                --cluster ${ECS_CLUSTER} \
                                --service-name ${ECS_SERVICE} \
                                --task-definition ${ECS_TASK_DEFINITION} \
                                --desired-count 1 \
                                --launch-type FARGATE \
                                --network-configuration "awsvpcConfiguration={subnets=[$SUBNETS],securityGroups=[$SECURITY_GROUP],assignPublicIp=ENABLED}"
                        fi
                        
                        # Get service URL (if using Application Load Balancer, otherwise this will show the task's public IP)
                        echo "Service deployed successfully!"
                        echo "Getting service details..."
                        aws ecs describe-services --cluster ${ECS_CLUSTER} --services ${ECS_SERVICE} --query 'services[0].serviceName' --output text
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            // Cleanup
            sh 'docker system prune -f || true'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}