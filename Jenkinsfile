pipeline {
    agent any
    environment {
        DOCKER_HUB_REPO = 'Abhishektu1/myflaskapp'  // Replace with your Docker Hub username/repo
    }
    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${DOCKER_HUB_REPO}:${BUILD_NUMBER} .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push ${DOCKER_HUB_REPO}:${BUILD_NUMBER}'
                }
            }
        }
        stage('Deploy to EC2') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'my-ec2-instance',
                            transfers: [
                                sshTransfer(
                                    execCommand: """
                                        docker pull ${DOCKER_HUB_REPO}:${BUILD_NUMBER}
                                        docker stop myflaskapp || true
                                        docker rm myflaskapp || true
                                        docker run -d --name myflaskapp -p 5000:5000 ${DOCKER_HUB_REPO}:${BUILD_NUMBER}
                                    """
                                )
                            ]
                        )
                    ]
                )
            }
        }
    }
}
