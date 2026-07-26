pipeline {
    agent any

    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: "1.0.${BUILD_NUMBER}", description: 'Docker image tag/version')
        booleanParam(name: 'PUSH_TO_HUB', defaultValue: true, description: 'Push image to Docker Hub')
    }

    environment {
        DOCKER_HUB_REPO = "shwetang95/jenkins-aws-app"
        DOCKER_HUB_CREDENTIALS = "docker-hub-creds_shway"
        IMAGE_TAG = "${params.DOCKER_IMAGE_TAG}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "=== Stage: Checkout ==="
                checkout scm
                sh 'ls -la'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "=== Stage: Build Docker Image ==="
                echo "Building image: ${DOCKER_HUB_REPO}:${IMAGE_TAG}"
                sh """
                    docker build -t ${DOCKER_HUB_REPO}:${IMAGE_TAG} .
                    docker build -t ${DOCKER_HUB_REPO}:latest .
                """
                echo "Docker image built successfully!"
            }
        }

        stage('Test Docker Image') {
            steps {
                echo "=== Stage: Test Docker Image ==="
                sh """
                    docker run -d --name test-container -p 5001:5000 ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                    sleep 3
                    curl -f http://localhost:5001/health || exit 1
                    echo "Health check passed!"
                    docker stop test-container
                    docker rm test-container
                """
            }
        }

        stage('Push to Docker Hub') {
            when {
                expression { return params.PUSH_TO_HUB == true }
            }
            steps {
                echo "=== Stage: Push to Docker Hub ==="
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_HUB_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                        docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKER_HUB_REPO}:latest
                        docker logout
                    """
                }
                echo "Pushed ${DOCKER_HUB_REPO}:${IMAGE_TAG} to Docker Hub!"
            }
        }

        stage('Cleanup') {
            steps {
                echo "=== Stage: Cleanup ==="
                sh """
                    docker rmi ${DOCKER_HUB_REPO}:${IMAGE_TAG} || true
                    docker rmi ${DOCKER_HUB_REPO}:latest || true
                """
                echo "Cleanup complete!"
            }
        }
    }

    post {
        success {
            echo """
            =========================================
              Pipeline SUCCESS
            =========================================
              Image: ${DOCKER_HUB_REPO}:${IMAGE_TAG}
              Docker Hub: https://hub.docker.com/r/${DOCKER_HUB_REPO}
            =========================================
            """
        }
        failure {
            echo "Pipeline FAILED - check logs above"
            sh 'docker stop test-container || true'
            sh 'docker rm test-container || true'
        }
    }
}
