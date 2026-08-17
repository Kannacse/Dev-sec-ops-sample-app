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

                    echo "Source verification successful."
                '''
            }
        }

        stage('Secret Scan') {
            steps {
                sh '''
                    echo "Running Gitleaks secret scan..."

                    gitleaks detect \
                        --source . \
                        --no-banner \
                        --verbose

                    echo "Gitleaks scan completed successfully."
                '''
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

        stage('Application Test') {
            steps {
                sh '''
                    echo "Running application syntax check..."

                    node --check app.js

                    echo "Application syntax check successful."
                '''
            }
        }
    }

    post {
        success {
            echo '========================================'
            echo 'Pipeline completed successfully.'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'Pipeline failed.'
            echo '========================================'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
