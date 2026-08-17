pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Source') {
            steps {
                sh '''
                    echo "Checking application files..."

                    ls -la

                    test -f app.js
                    test -f package.json
                    test -f Dockerfile
                    test -f sonar-project.properties

                    echo "Source verification successful."
                '''
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    echo "Running Gitleaks..."

                    gitleaks detect \
                        --source . \
                        --no-banner

                    echo "Gitleaks scan completed successfully."
                '''
            }
        }

        stage('SAST - SonarQube') {
            steps {
                withSonarQubeEnv('sonarqube-dev-sec') {
                    sh '''
                        echo "Running SonarQube analysis..."

                        sonar-scanner

                        echo "SonarQube analysis completed."
                    '''
                }
            }
        }

        stage('Dependency Audit') {
            steps {
                sh '''
                    echo "Running npm dependency audit..."

                    npm audit --audit-level=high

                    echo "Dependency audit completed."
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Installing dependencies..."

                    npm install

                    echo "Dependencies installed successfully."
                '''
            }
        }

        stage('Application Test') {
            steps {
                sh '''
                    echo "Running Node.js syntax check..."

                    node --check app.js

                    echo "Application test successful."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                        -t devsecops-sample-app:${BUILD_NUMBER} .

                    echo "Docker image built successfully."
                '''
            }
        }

        stage('Container Scan - Trivy') {
            steps {
                sh '''
                    echo "Running Trivy container scan..."

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        devsecops-sample-app:${BUILD_NUMBER}

                    echo "Trivy scan completed successfully."
                '''
            }
        }
    }

    post {
        success {
            echo '=========================================='
            echo 'DevSecOps pipeline completed successfully'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo 'DevSecOps pipeline FAILED'
            echo '=========================================='
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
