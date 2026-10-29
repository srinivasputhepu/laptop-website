
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 15, unit: 'MINUTES')
    }

    environment {
        IMAGE_NAME     = 'srinivasputhepu/laptop-website'
        EC2_HOST       = '13.235.74.228'
        EC2_USER       = 'ubuntu'
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

                    echo "Checking website files..."
                    test -f index.html

                    echo "Checking HTML content..."
                    grep -qi "<html" index.html
                    grep -qi "<body" index.html

                    echo "Website tests passed."
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
                            -u "$DOCKER_USER" \
                            --password-stdin

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

                        echo "Deploying build ${BUILD_NUMBER} to EC2..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=30 \
                            ${EC2_USER}@${EC2_HOST} \
                            "BUILD_NUMBER=${BUILD_NUMBER} bash -s" <<'REMOTE'

set -eu

IMAGE_NAME='srinivasputhepu/laptop-website'
CONTAINER_NAME='laptop-website'
NEW_IMAGE="${IMAGE_NAME}:${BUILD_NUMBER}"

echo "Recording currently deployed image..."

PREVIOUS_IMAGE=$(docker inspect \
    --format='{{.Config.Image}}' \
    "$CONTAINER_NAME" 2>/dev/null || true)

echo "Previous image: ${PREVIOUS_IMAGE:-none}"
echo "Target image: $NEW_IMAGE"

echo "Pulling target image..."
docker pull "$NEW_IMAGE"

echo "Stopping the existing container..."
docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true

echo "Starting the new container..."

if ! docker run -d \
    --name "$CONTAINER_NAME" \
    --restart unless-stopped \
    -p 80:80 \
    "$NEW_IMAGE"; then

    echo "New container failed to start."

    if [ -n "$PREVIOUS_IMAGE" ]; then
        echo "Attempting rollback to $PREVIOUS_IMAGE..."

        docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true

        if docker run -d \
            --name "$CONTAINER_NAME" \
            --restart unless-stopped \
            -p 80:80 \
            "$PREVIOUS_IMAGE"; then
            echo "Previous container restarted."
        else
            echo "ERROR: Rollback container could not start."
        fi
    else
        echo "No previous image is available."
    fi

    exit 1
fi

echo "Checking website health..."

HEALTH_OK=false

for attempt in 1 2 3 4 5; do
    if curl -fsS --max-time 5 http://localhost/ >/dev/null; then
        echo "Deployment health check passed."
        HEALTH_OK=true
        break
    fi

    echo "Health check attempt ${attempt} failed."
    sleep 5
done

if [ "$HEALTH_OK" = true ]; then
    echo "Deployment successful: $NEW_IMAGE"
    exit 0
fi

echo "Health check failed. Rolling back..."

docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true

if [ -n "$PREVIOUS_IMAGE" ]; then
    if docker run -d \
        --name "$CONTAINER_NAME" \
        --restart unless-stopped \
        -p 80:80 \
        "$PREVIOUS_IMAGE"; then

        echo "Rollback container started."

        ROLLBACK_OK=false

        for attempt in 1 2 3 4 5; do
            if curl -fsS --max-time 5 http://localhost/ >/dev/null; then
                echo "Rollback health check passed."
                ROLLBACK_OK=true
                break
            fi

            echo "Rollback health check attempt ${attempt} failed."
            sleep 5
        done

        if [ "$ROLLBACK_OK" != true ]; then
            echo "ERROR: Rollback container is running but unhealthy."
        fi
    else
        echo "ERROR: Rollback container could not start."
    fi
else
    echo "ERROR: No previous image is available for rollback."
fi

# Fail the Jenkins build even if rollback succeeds.
exit 1

REMOTE
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }

        success {
            echo 'SUCCESS: Website built, pushed, deployed, and verified.'
        }

        failure {
            echo 'FAILURE: Check Console Output. A rollback may have been attempted.'
        }
    }
}
