pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t srinivasputhepu/laptop-website:${env.BUILD_NUMBER} \
                    -t srinivasputhepu/laptop-website:latest .
                """
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
                    sh """
                        echo "\$DOCKER_TOKEN" | docker login \
                        -u "\$DOCKER_USER" \
                        --password-stdin

                        docker push srinivasputhepu/laptop-website:${env.BUILD_NUMBER}

                        docker push srinivasputhepu/laptop-website:latest

                        docker logout
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline Finished'
        }

        success {
            echo 'Docker image successfully pushed to Docker Hub'
        }

        failure {
            echo 'Pipeline Failed'
        }
    }
}
