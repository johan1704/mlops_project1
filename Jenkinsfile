pipeline{
    agent any
    
    environment {
        VENV_DIR = 'venv'
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
    }
}