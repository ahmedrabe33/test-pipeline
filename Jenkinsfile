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

                    writeFile file: 'build-info.env', text: """
IMAGE_TAG=${shortCommit}-${BUILD_NUMBER}
IMAGE_FULL_NAME=${DOCKERHUB_USERNAME}/${APP_NAME}:${shortCommit}-${BUILD_NUMBER}
IMAGE_REPO=${DOCKERHUB_USERNAME}/${APP_NAME}
APP_NAME=${APP_NAME}
BUILD_NUMBER=${BUILD_NUMBER}
GIT_COMMIT_SHORT=${shortCommit}
"""
                }

                sh '''
                    echo "Current workspace:"
                    pwd

                    echo "Repository files:"
                    ls -la

                    echo "Build metadata:"
                    cat build-info.env

                    echo "Verify required files:"
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
                    . ./build-info.env

                    echo "Building Docker image:"
                    echo "$IMAGE_FULL_NAME"

                    docker build \
                      -t "$IMAGE_FULL_NAME" \
                      -t "$IMAGE_REPO:latest" \
                      .

                    echo "Docker images:"
                    docker images | grep "$APP_NAME"
                '''
            }
        }

        stage('Image Smoke Test') {
            agent { label 'docker-agent' }

            steps {
                sh '''
                    . ./build-info.env

                    echo "Running smoke test for:"
                    echo "$IMAGE_FULL_NAME"

                    docker rm -f "${APP_NAME}-test" 2>/dev/null || true

                    docker run -d \
                      --name "${APP_NAME}-test" \
                      -p 3000:3000 \
                      "$IMAGE_FULL_NAME"

                    sleep 5

                    curl -f http://localhost:3000/health

                    docker logs "${APP_NAME}-test"

                    docker rm -f "${APP_NAME}-test" 2>/dev/null || true
                '''
            }
        }

        stage('Trivy Image Scan') {
            agent { label 'docker-agent' }

            steps {
                sh '''
                    . ./build-info.env

                    echo "Scanning image:"
                    echo "$IMAGE_FULL_NAME"

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --ignore-unfixed \
                      "$IMAGE_FULL_NAME"
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
                        . ./build-info.env

                        echo "Logging in to DockerHub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "Pushing image:"
                        echo "$IMAGE_FULL_NAME"

                        docker push "$IMAGE_FULL_NAME"
                        docker push "$IMAGE_REPO:latest"
                    '''
                }
            }
        }

        stage('Archive Build Metadata') {
            agent { label 'test-agent' }

            steps {
                unstash 'source-code'

                sh '''
                    echo "Final build metadata:"
                    cat build-info.env
                '''

                archiveArtifacts artifacts: 'build-info.env', fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Check console output."
        }

        always {
            node('docker-agent') {
                sh '''
                    echo "Cleaning Docker resources..."
                    docker rm -f test-pipeline-app-test 2>/dev/null || true
                    docker image prune -f || true
                '''
            }
        }
    }
}