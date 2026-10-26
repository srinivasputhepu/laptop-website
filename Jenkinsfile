
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        IMAGE_NAME = 'srinivasputhepu/laptop-website'
        EC2_HOST = '15.206.194.54'
        EC2_USER = 'ubuntu'
        CONTAINER_NAME = 'laptop-website'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/srinivasputhepu/laptop-website.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                        -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ['aws-ec2-key']) {
                    sh '''
                        set -eu

                        IMAGE_TAG="${IMAGE_NAME}:${BUILD_NUMBER}"

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=120 \
                            "${EC2_USER}@${EC2_HOST}" \
                            "docker pull ${IMAGE_TAG} &&
                             (docker rm -f ${CONTAINER_NAME} || true) &&
                             docker run -d
                               --name ${CONTAINER_NAME}
                               --restart unless-stopped
                               -p 80:80
                               ${IMAGE_TAG}"
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'Image built, pushed, and deployed to EC2 successfully.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
