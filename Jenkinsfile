// Define environment variables for the pipeline
environment {
    // ID of Docker Hub credentials stored in Jenkins
    DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
    // Docker Hub repository name (replace with yourusername/your-repo)
    DOCKER_IMAGE = 'abhishektu/myapp'
    // Short Git commit hash for image tagging
    GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
}

// Pipeline stages
stages {
    // Stage 1: Checkout code from GitHub
    stage('Checkout Code') {
        steps {
            // Pull the latest code from the specified GitHub repository and branch
            git branch: 'main', url: 'https://github.com/Abhishektu1/old.git'
        }
    }

    // Stage 2: Build the Docker image
    stage('Build Docker Image') {
        steps {
            script {
                // Build Docker image with the Git commit hash as the tag
                sh "docker build -t ${DOCKER_IMAGE}:${GIT_COMMIT} ."
                // Also tag the image as 'latest' for convenience
                sh "docker tag ${DOCKER_IMAGE}:${GIT_COMMIT} ${DOCKER_IMAGE}:latest"
            }
        }
    }

    // Stage 3: Push the Docker image to Docker Hub
    stage('Push to Docker Hub') {
        steps {
            script {
                // Use stored credentials to log in to Docker Hub
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    // Log in to Docker Hub using credentials
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    // Push the image with the commit hash tag
                    sh "docker push ${DOCKER_IMAGE}:${GIT_COMMIT}"
                    // Push the image with the 'latest' tag
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }
    }
}

// Post-build actions
post {
    // Always clean up local Docker images to save space
    always {
        sh "docker rmi ${DOCKER_IMAGE}:${GIT_COMMIT} || true"
        sh "docker rmi ${DOCKER_IMAGE}:latest || true"
    }
    // Log success message if the pipeline completes
    success {
        echo 'Docker image successfully built and pushed to Docker Hub!'
    }
    // Log failure message if the pipeline fails
    failure {
        echo 'Failed to build or push the Docker image.'
    }
}
