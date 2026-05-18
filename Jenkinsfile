pipeline {
    agent none

    environment {
        APP_NAME = "test-pipeline-app"
        DOCKERHUB_USERNAME = "ahmedrabie222000"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${APP_NAME}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            agent { label 'test-agent' }

            steps {
                checkout scm

                sh '''
                    echo "Current workspace:"
                    pwd

                    echo "Repository files:"
                    ls -la

                    echo "Git commit:"
                    git rev-parse --short HEAD
                '''

                stash name: 'source-code', includes: '**/*'
            }
        }

        stage('Parallel Code Validation') {
            parallel {
                stage('Install Dependencies and Unit Tests') {
                    agent { label 'test-agent' }

                    steps {
                        unstash 'source-code'

                        sh '''
                            echo "Installing dependencies..."
                            npm ci

                            echo "Running unit tests..."
                            npm test
                        '''
                    }
                }

                stage('Lint Code') {
                    agent { label 'test-agent' }

                    steps {
                        unstash 'source-code'

                        sh '''
                            echo "Installing dependencies..."
                            npm ci

                            echo "Running lint..."
                            npm run lint
                        '''
                    }
                }

                stage('Dependency Audit') {
                    agent { label 'test-agent' }

                    steps {
                        unstash 'source-code'

                        sh '''
                            echo "Installing dependencies..."
                            npm ci

                            echo "Running npm audit..."
                            npm audit --audit-level=high || true
                        '''
                    }
                }
            }
        }

        stage('Docker Build') {
            agent { label 'docker-agent' }

            steps {
                unstash 'source-code'

                sh '''
                    echo "Building Docker image..."
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .

                    echo "Tagging image as latest..."
                    docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest

                    echo "Docker images:"
                    docker images | grep $APP_NAME
                '''
            }
        }

        stage('Image Smoke Test') {
            agent { label 'docker-agent' }

            steps {
                sh '''
                    echo "Running container smoke test..."

                    docker rm -f $APP_NAME-test || true

                    docker run -d \
                      --name $APP_NAME-test \
                      -p 3000:3000 \
                      $IMAGE_NAME:$IMAGE_TAG

                    sleep 5

                    curl -f http://localhost:3000/health

                    docker logs $APP_NAME-test

                    docker rm -f $APP_NAME-test
                '''
            }
        }

        stage('Trivy Image Scan') {
            agent { label 'docker-agent' }

            steps {
                sh '''
                    echo "Scanning Docker image with Trivy..."

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Push Docker Image') {
            agent { label 'docker-agent' }

            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "Logging in to DockerHub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "Pushing image tag..."
                        docker push $IMAGE_NAME:$IMAGE_TAG

                        echo "Pushing latest tag..."
                        docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Image pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo "Pipeline failed. Check console output."
        }

        always {
            node('docker-agent') {
                sh '''
                    docker rm -f $APP_NAME-test || true
                    docker image prune -f || true
                '''
            }
        }
    }
}