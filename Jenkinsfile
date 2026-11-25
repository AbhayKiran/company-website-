pipeline {
    // Every stage runs on the build-server agent, never on the Jenkins controller.
    agent { label 'build-server' }

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // ---- REPLACE THESE PLACEHOLDERS (see README.md) ----
        DOCKERHUB_USERNAME         = 'DOCKERHUB_USERNAME'
        DOCKERHUB_REPOSITORY       = 'DOCKERHUB_REPOSITORY'
        DOCKERHUB_CREDENTIAL_ID    = 'DOCKERHUB_CREDENTIAL_ID'
        DEPLOYMENT_SERVER_IP       = 'DEPLOYMENT_SERVER_IP'
        DEPLOYMENT_SSH_CREDENTIAL_ID = 'DEPLOYMENT_SSH_CREDENTIAL_ID'
        GITHUB_REPOSITORY_URL      = 'GITHUB_REPOSITORY_URL'
        // -----------------------------------------------------

        IMAGE_NAME     = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPOSITORY}"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_NAME = 'company-website'
        HOST_PORT      = '9090'
    }

    stages {

        stage('Fetch GitHub Repository') {
            steps {
                git url: "${GITHUB_REPOSITORY_URL}", branch: 'main'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t "$IMAGE_NAME:$IMAGE_TAG" -t "$IMAGE_NAME:latest" .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIAL_ID}",
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh '''
                    docker push "$IMAGE_NAME:$IMAGE_TAG"
                    docker push "$IMAGE_NAME:latest"
                '''
            }
        }

        stage('Deploy on Deployment Server') {
            steps {
                // The build server only opens an SSH session.
                // The container runs on the deployment server.
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${DEPLOYMENT_SSH_CREDENTIAL_ID}",
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        SSH_OPTS="-o StrictHostKeyChecking=accept-new -o BatchMode=yes -i $SSH_KEY"
                        TARGET="$SSH_USER@$DEPLOYMENT_SERVER_IP"

                        ssh $SSH_OPTS "$TARGET" "docker pull $IMAGE_NAME:$IMAGE_TAG"

                        ssh $SSH_OPTS "$TARGET" "docker stop $CONTAINER_NAME || true"
                        ssh $SSH_OPTS "$TARGET" "docker rm $CONTAINER_NAME || true"

                        ssh $SSH_OPTS "$TARGET" "docker run -d --name $CONTAINER_NAME --restart unless-stopped -p $HOST_PORT:80 $IMAGE_NAME:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${DEPLOYMENT_SSH_CREDENTIAL_ID}",
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        SSH_OPTS="-o StrictHostKeyChecking=accept-new -o BatchMode=yes -i $SSH_KEY"
                        TARGET="$SSH_USER@$DEPLOYMENT_SERVER_IP"

                        ssh $SSH_OPTS "$TARGET" "docker ps --filter name=$CONTAINER_NAME --filter status=running --format '{{.Names}} {{.Status}}'"

                        for i in 1 2 3 4 5; do
                            if ssh $SSH_OPTS "$TARGET" "curl -fsS http://localhost:$HOST_PORT > /dev/null"; then
                                echo "Deployment verified: application is responding on port $HOST_PORT."
                                exit 0
                            fi
                            echo "Waiting for application to start (attempt $i/5)..."
                            sleep 5
                        done

                        echo "Deployment verification failed."
                        exit 1
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'docker image prune -f || true'
        }
        success {
            echo "Application deployed: http://${DEPLOYMENT_SERVER_IP}:${HOST_PORT}"
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }
    }
}
