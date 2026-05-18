pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        APP_NAME = "test-pipeline-app"
        DOCKERHUB_USERNAME = "ahmedrabie222000"
        IMAGE_REPO = "${DOCKERHUB_USERNAME}/${APP_NAME}"
        IMAGE_TAG = ""
        IMAGE_FULL_NAME = ""
    }

    stages {
        stage('Checkout Source Code') {
            agent { label 'test-agent' }

            steps {
                checkout scm

                script {
                    def shortCommit = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${shortCommit}-${BUILD_NUMBER}"
                    env.IMAGE_FULL_NAME = "${env.IMAGE_REPO}:${env.IMAGE_TAG}"
                }

                sh '''
                    echo "Current workspace:"
                    pwd

                    echo "Repository files:"
                    ls -la

                    echo "Git commit:"
                    git rev-parse --short HEAD

                    echo "Image tag:"
                    echo "$IMAGE_TAG"

                    echo "Verifying required files..."
                    test -f package.json
                    test -f package-lock.json
                    test -f Dockerfile
                    test -d src
                    test -d test
                '''

                stash name: 'source-code', includes: '**/*'
            }
        }

        stage('Parallel Code Quality') {
            parallel {
                stage('Unit Tests') {
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

                            echo "Running dependency audit..."
                            npm audit --audit-level=high
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
                    docker build \
                      -t $IMAGE_FULL_NAME \
                      -t $IMAGE_REPO:latest \
                      .

                    echo "Docker image built:"
                    docker images | grep $APP_NAME
                '''
            }
        }

        stage('Image Smoke Test') {
            agent { label 'docker-agent' }

            steps {
                sh '''
                    echo "Running container smoke test..."

                    docker rm -f ${APP_NAME}-test 2>/dev/null || true

                    docker run -d \
                      --name ${APP_NAME}-test \
                      -p 3000:3000 \
                      $IMAGE_FULL_NAME

                    sleep 5

                    echo "Testing health endpoint..."
                    curl -f http://localhost:3000/health

                    echo "Container logs:"
                    docker logs ${APP_NAME}-test

                    docker rm -f ${APP_NAME}-test 2>/dev/null || true
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
                      --exit-code 1 \
                      --ignore-unfixed \
                      $IMAGE_FULL_NAME
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

                        echo "Pushing version tag..."
                        docker push $IMAGE_FULL_NAME

                        echo "Pushing latest tag..."
                        docker push $IMAGE_REPO:latest
                    '''
                }
            }
        }

        stage('Archive Build Metadata') {
            agent { label 'test-agent' }

            steps {
                sh '''
                    cat > build-info.txt <<EOF
APP_NAME=$APP_NAME
IMAGE_REPO=$IMAGE_REPO
IMAGE_TAG=$IMAGE_TAG
IMAGE_FULL_NAME=$IMAGE_FULL_NAME
BUILD_NUMBER=$BUILD_NUMBER
GIT_COMMIT=$(git rev-parse HEAD 2>/dev/null || echo unknown)
EOF

                    cat build-info.txt
                '''

                archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Image pushed: ${IMAGE_FULL_NAME}"
        }

        failure {
            echo "Pipeline failed. Check console output."
        }

        always {
            node('docker-agent') {
                sh '''
                    echo "Cleaning Docker resources..."
                    docker rm -f ${APP_NAME}-test 2>/dev/null || true
                    docker image prune -f || true
                '''
            }
        }
    }
}