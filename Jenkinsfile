pipeline {
    // No global agent: each stage group picks its own agent.
    // Nothing runs on the Jenkins controller.
    agent none

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // ---- REPLACE THESE PLACEHOLDERS (see README.md) ----
        DOCKERHUB_USERNAME      = 'abhaykiran'
        DOCKERHUB_REPOSITORY    = 'company-website'
        DOCKERHUB_CREDENTIAL_ID = 'dockerhub-credentials'
        DEPLOYMENT_SERVER_IP    = '52.34.211.206'
        GITHUB_REPOSITORY_URL   = 'https://github.com/AbhayKiran/company-website-.git'
        // -----------------------------------------------------

        IMAGE_NAME     = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPOSITORY}"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_NAME = 'company-website'
        HOST_PORT      = '9090'
    }

    stages {

        // ================= BUILD SERVER =================
        stage('Build Server Operations') {
            agent { label 'build-server' }

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
            }

            post {
                always {
                    sh 'docker logout || true'
                    sh 'docker image prune -f || true'
                }
            }
        }

        // ================= DEPLOYMENT SERVER =================
        stage('Deployment Server Operations') {
            agent { label 'deploy-agent' }

            stages {

                stage('Pull Image on Deployment Server') {
                    steps {
                        // Login is only needed if the Docker Hub repository is private.
                        withCredentials([usernamePassword(
                            credentialsId: "${DOCKERHUB_CREDENTIAL_ID}",
                            usernameVariable: 'DH_USER',
                            passwordVariable: 'DH_PASS'
                        )]) {
                            sh '''
                                echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                                docker pull "$IMAGE_NAME:$IMAGE_TAG"
                            '''
                        }
                    }
                }

                stage('Replace Running Container') {
                    steps {
                        sh '''
                            docker stop "$CONTAINER_NAME" || true
                            docker rm "$CONTAINER_NAME" || true

                            docker run -d \
                                --name "$CONTAINER_NAME" \
                                --restart unless-stopped \
                                -p "$HOST_PORT":80 \
                                "$IMAGE_NAME:$IMAGE_TAG"
                        '''
                    }
                }

                stage('Verify Deployment') {
                    steps {
                        sh '''
                            docker ps --filter "name=$CONTAINER_NAME" --filter "status=running" \
                                --format '{{.Names}} {{.Status}}'

                            for i in 1 2 3 4 5; do
                                if curl -fsS "http://localhost:$HOST_PORT" > /dev/null; then
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

            post {
                always {
                    sh 'docker logout || true'
                }
                success {
                    echo "Application deployed: http://${DEPLOYMENT_SERVER_IP}:${HOST_PORT}"
                }
                failure {
                    echo 'Deployment failed. Check the stage logs above.'
                }
            }
        }
    }
}
