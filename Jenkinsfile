
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



	stage('Deploy and Verify') {
	    steps {
	        sshagent(credentials: ['aws-ec2-key']) {
	            sh '''
	                set -eu

	                echo "Starting deployment..."

	                ssh \
	                    -o StrictHostKeyChecking=no \
	                    -o ConnectTimeout=120 \
	                    ubuntu@${EC2_HOST} \
	                    "BUILD_NUMBER=${BUILD_NUMBER} bash -s" <<'REMOTE'
	                set -eu

	                IMAGE_NAME='srinivasputhepu/laptop-website'
	                NEW_IMAGE="${IMAGE_NAME}:${BUILD_NUMBER}"
	                CONTAINER_NAME='laptop-website'

	                echo "Recording the current image..."

	                PREVIOUS_IMAGE=$(docker inspect \
	                    --format='{{.Config.Image}}' \
	                    "$CONTAINER_NAME" 2>/dev/null || true)

	                echo "Previous image: ${PREVIOUS_IMAGE:-none}"
	                echo "New image: $NEW_IMAGE"

	                echo "Pulling new image..."
	                docker pull "$NEW_IMAGE"

	                echo "Replacing the container..."
	                docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true

	                if ! docker run -d \
	                    --name "$CONTAINER_NAME" \
	                    --restart unless-stopped \
	                    -p 80:80 \
	                    "$NEW_IMAGE"; then

	                    echo "New container failed to start."
	                    if [ -n "$PREVIOUS_IMAGE" ]; then
	                        echo "Rolling back to $PREVIOUS_IMAGE..."
	                        docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
	                        docker run -d \
	                            --name "$CONTAINER_NAME" \
	                            --restart unless-stopped \
	                            -p 80:80 \
	                            "$PREVIOUS_IMAGE"
	                    fi
	                    exit 1
	                fi

	                echo "Checking website health..."
			exit 1

	                for attempt in 1 2 3 4 5; do
	                    if curl -fsS http://localhost/ >/dev/null; then
	                        echo "Deployment health check passed!"
	                        exit 0
	                    fi

	                    echo "Attempt $attempt failed. Retrying..."
	                    sleep 5
	                done

	                echo "Simulating health-check failure for rollback test."
				echo "Health check failed. Rolling back..."

				docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true

				if [ -n "$PREVIOUS_IMAGE" ]; then
				    docker run -d \
				        --name "$CONTAINER_NAME" \
				        --restart unless-stopped \
				        -p 80:80 \
				        "$PREVIOUS_IMAGE"

				    echo "Rollback container started."
				else
				    echo "No previous image available for rollback."
				fi

				exit 1
	                REMOTE
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
