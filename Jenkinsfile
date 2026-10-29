
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
        EC2_HOST = '13.235.74.228'
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

	
stage('Test Website') {
    steps {
        sh '''
            set -eu

            echo "Checking whether index.html exists..."

            test -f index.html

            echo "Checking HTML content..."

            grep -qi "<html" index.html
            grep -qi "<body" index.html

            echo "website tests passed"
        '''
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

                ssh \
                    -o StrictHostKeyChecking=no \
                    -o ConnectTimeout=120 \
                    ubuntu@3.110.166.125 \
                    "set -e;
                     docker pull srinivasputhepu/laptop-website:${BUILD_NUMBER};
                     docker rm -f laptop-website >/dev/null 2>&1 || true;
                     docker run -d --name laptop-website --restart unless-stopped -p 80:80 srinivasputhepu/laptop-website:${BUILD_NUMBER}"
            '''
        }
    }
}


stage('Deployment Health Check') {
    steps {
        sshagent(credentials: ['aws-ec2-key']) {
            sh '''
                set -eu

                echo "Checking website health on EC2..."

                for attempt in 1 2 3 4 5; do
                    if ssh \
                        -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=10 \
                        ubuntu@${EC2_HOST} \
                        "curl -fsS http://localhost/ > /dev/null"; then

                        echo "Health check passed!"
                        exit 0
                    fi

                    echo "Attempt ${attempt} failed. Retrying..."
                    sleep 5
                done

                echo "Health check failed after 5 attempts."
                exit 1
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
