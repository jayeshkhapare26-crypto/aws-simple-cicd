pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'YOUR_DOCKERHUB_USERNAME/aws-simple-cicd'
        APP_SERVER = 'ubuntu@YOUR_APP_SERVER_PUBLIC_IP'
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

        stage('Test') {
            steps {
                sh 'test -s index.html'
                sh 'grep -q "My First AWS CI/CD Project" index.html'
                echo 'Basic application test passed'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKERHUB_REPO:$BUILD_NUMBER .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin
                        docker push "$DOCKERHUB_REPO:$BUILD_NUMBER"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to AWS EC2') {
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

                        ssh -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=accept-new \
                          "$APP_SERVER" \
                          "sudo docker pull $DOCKERHUB_REPO:$BUILD_NUMBER &&
                           sudo docker rm -f webapp 2>/dev/null || true"
                    '''

                    sh '''
                        ssh -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=accept-new \
                          "$APP_SERVER" \
                          "sudo docker run -d --name webapp \
                           --restart unless-stopped -p 8080:80 \
                           $DOCKERHUB_REPO:$BUILD_NUMBER"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the Jenkins console log.'
        }
    }
}