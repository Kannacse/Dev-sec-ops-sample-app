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
                    echo "=========================================="
                    echo "Verifying Application Source"
                    echo "=========================================="

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
                    echo "=========================================="
                    echo "Running Gitleaks Secret Scan"
                    echo "=========================================="

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
                            exit 1
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
                            echo "Running SonarQube SAST Analysis"
                            echo "=========================================="

                            sonar-scanner

                            echo "SonarQube analysis completed successfully."
                        '''
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Installing Application Dependencies"
                    echo "=========================================="

                    npm install

                    echo "Dependencies installed successfully."
                '''
            }
        }

        stage('Dependency Audit') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running NPM Dependency Audit"
                    echo "=========================================="

                    npm audit --audit-level=high || true

                    echo "WARNING: NPM audit completed."
                    echo "Review dependency vulnerabilities before production deployment."
                '''
            }
        }

        stage('Application Test') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Application Test"
                    echo "=========================================="

                    node --check app.js

                    echo "Application syntax check successful."
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
                    echo "Running Trivy Container Security Scan"
                    echo "=========================================="

                    echo "Scanning image:"
                    echo "devsecops-sample-app:${BUILD_NUMBER}"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        devsecops-sample-app:${BUILD_NUMBER} || true

                    echo ""
                    echo "WARNING: Trivy scan completed."
                    echo "HIGH/CRITICAL vulnerabilities may require remediation."
                    echo "Pipeline will continue for this development environment."
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
