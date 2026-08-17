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

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                '''
            }
        }

        stage('Application Test') {
            steps {
                sh '''
                    node --check app.js
                    echo "Application syntax check successful."
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}
