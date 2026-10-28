pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '290027677515'
        ECR_REPOSITORY = 'aws-simple-cicd'
        APP_SERVER = 'ubuntu@65.1.134.179'
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify AWS Identity') {
            steps {
                sh '''
                    aws --version
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    test -s index.html
                    grep -q "My First AWS CI/CD Project" index.html
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
                    docker build -t "${ECR_URI}:${BUILD_NUMBER}" .
                '''
            }
        }

        stage('Login and Push to ECR') {
            steps {
                sh '''
                    set -eu
                    ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"

                    aws ecr get-login-password --region "$AWS_REGION" \
                      | docker login \
                          --username AWS \
                          --password-stdin \
                          "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    docker push "${ECR_URI}:${BUILD_NUMBER}"
                    docker logout "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                '''
            }
        }

        stage('Deploy to App Server') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'app-server-key',
                        variable: 'SSH_KEY'
                    )
                ]) {
                    sh '''
                        set -eu
                        chmod 600 "$SSH_KEY"

                        ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"

                        ssh -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=accept-new \
                          "$APP_SERVER" \
                          "aws ecr get-login-password --region ${AWS_REGION} | sudo docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com && sudo docker pull ${ECR_URI}:${BUILD_NUMBER} && (sudo docker rm -f webapp || true) && sudo docker run -d --name webapp --restart unless-stopped -p 8080:80 ${ECR_URI}:${BUILD_NUMBER}"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    curl -fsS --retry 5 --retry-delay 2 \
                      "http://${APP_SERVER#ubuntu@}:8080/" \
                      | grep -q "My First AWS CI/CD Project"
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Image pushed to ECR and deployed to App Server.'
        }
        failure {
            echo 'FAILED: Check Console Output for the failing stage.'
        }
    }
}