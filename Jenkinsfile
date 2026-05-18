pipeline {
    agent { label 'build-agent' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    echo "Workspace:"
                    pwd
                    ls -la
                '''
            }
        }

        stage('Validate Tools') {
            steps {
                sh '''
                    echo "Checking tools on Jenkins agent..."
                    git --version
                    helm version
                    kubectl version --client
                '''
            }
        }

        stage('Helm Lint') {
            steps {
                sh '''
                    helm lint helm-chart
                '''
            }
        }

        stage('Render Kubernetes Manifests') {
            steps {
                sh '''
                    helm template online-boutique helm-chart > rendered-manifests.yaml
                    ls -lh rendered-manifests.yaml
                '''
            }
        }

        stage('Kubernetes Dry Run') {
            steps {
                sh '''
                    kubectl apply --dry-run=client -f rendered-manifests.yaml
                '''
            }
        }

        stage('Summary') {
            steps {
                echo "Online Boutique CI validation completed successfully."
            }
        }
    }
}
