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

        stage('SonarQube Connection Test') {
            steps {
                withSonarQubeEnv('sonarqube-dev-sec') {
                    sh '''
                        echo "=========================================="
                        echo "SonarQube Connection Test"
                        echo "=========================================="

                        echo "SonarQube URL: $SONAR_HOST_URL"

                        if [ -n "$SONAR_AUTH_TOKEN" ]; then
                            echo "Sonar authentication token is configured: YES"
                        else
                            echo "Sonar authentication token is configured: NO"
                        fi
                    '''
                }
            }
        }

        stage('SAST - SonarQube') {
    steps {
        withSonarQubeEnv('sonarqube-dev-sec') {
            withEnv([
                "PATH+SONAR=/opt/sonar-scanner/bin",
                "SONAR_TOKEN=${SONAR_AUTH_TOKEN}"
            ]) {
                sh '''
                    echo "=========================================="
                    echo "Running SonarQube Analysis"
                    echo "=========================================="

                    sonar-scanner

                    echo "SonarQube analysis completed successfully."
                '''
            }
        }
    }
}

stage('SAST - SonarQube') {
            steps {
                withSonarQubeEnv('sonarqube-dev-sec') {
                    withEnv(["PATH+SONAR=/opt/sonar-scanner/bin"]) {
                        sh '''
                            echo "=========================================="
                            echo "Running SonarQube Analysis"
                            echo "=========================================="

                            sonar-scanner

                            echo "SonarQube analysis completed."
                        '''
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Installing application dependencies..."

                    npm install

                    echo "Dependencies installed successfully."
                '''
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
                    echo "=========================================="
                    echo "Building Docker Image"
                    echo "=========================================="

                    docker build \
                        -t devsecops-sample-app:${BUILD_NUMBER} .

                    echo "Docker image built successfully."

                    docker images | grep devsecops-sample-app
                '''
            }
        }

        stage('Container Scan - Trivy') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Trivy Container Scan"
                    echo "=========================================="

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
            echo 'DevSecOps Pipeline Completed Successfully'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo 'DevSecOps Pipeline FAILED'
            echo '=========================================='
        }

        always {
            echo '=========================================='
            echo 'Pipeline execution completed.'
            echo '=========================================='
        }
    }
}
