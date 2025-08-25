pipeline{
    agent any

    stages{
        stage('cloning github repo to jenkins'){
            steps{
                script{
                    echo 'cloning github repo ..'
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/johan1704/mlops_project1.git']])
                }
            }
        }
    }
}