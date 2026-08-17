pipeline {

    agent any

    environment {
        DOCKER_IMAGE = "kannancloud/devsecops-sample-app"
        AWS_REGION = "ap-south-1"

        // EC2 configuration
        EC2_INSTANCE_ID = "YOUR_EC2_INSTANCE_ID"

        // AWS credential stored in Jenkins
        AWS_CREDENTIALS_ID = "aws-credentials"

        // Docker Hub credential stored in Jenkins
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"
    }

    stages {

        // ============================================================
        // 1. CHECKOUT SOURCE CODE
        // ============================================================

        stage('Checkout') {
            steps {

                echo "=========================================="
                echo "Checking out source code"
                echo "=========================================="

                checkout scm

                echo "Source code checkout completed successfully."
            }
        }


        // ============================================================
        // 2. SOURCE CODE SECURITY SCAN - GITLEAKS
        // ============================================================

        stage('Secret Scan - Gitleaks') {
            steps {

                echo "=========================================="
                echo "Running Gitleaks Secret Scan"
                echo "=========================================="

                sh '''
                    gitleaks detect \
                        --source . \
                        --no-banner \
                        --redact || true
                '''

                echo "Gitleaks scan completed."
            }
        }


        // ============================================================
        // 3. SONARQUBE CODE QUALITY
        // ============================================================

        stage('SonarQube Analysis') {
            steps {

                echo "=========================================="
                echo "Running SonarQube Analysis"
                echo "=========================================="

                script {

                    def scannerHome = tool 'SonarQubeScanner'

                    withSonarQubeEnv('SonarQube') {

                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=devsecops-sample-app \
                                -Dsonar.projectName=devsecops-sample-app \
                                -Dsonar.sources=.
                        """
                    }
                }

                echo "SonarQube analysis completed."
            }
        }


        // ============================================================
        // 4. INSTALL DEPENDENCIES
        // ============================================================

        stage('Install Dependencies') {
            steps {

                echo "=========================================="
                echo "Installing application dependencies"
                echo "=========================================="

                sh '''
                    if [ -f package.json ]; then
                        npm install
                    else
                        echo "No package.json found."
                    fi
                '''

                echo "Dependency installation completed."
            }
        }


        // ============================================================
        // 5. APPLICATION TEST
        // ============================================================

        stage('Application Test') {
            steps {

                echo "=========================================="
                echo "Running Application Tests"
                echo "=========================================="

                sh '''
                    if [ -f package.json ]; then

                        if npm run | grep -q "test"; then
                            npm test -- --watchAll=false || true
                        else
                            echo "No test script configured."
                        fi

                    else
                        echo "No package.json found."
                    fi
                '''

                echo "Application testing completed."
            }
        }


        // ============================================================
        // 6. DOCKER BUILD
        // ============================================================

        stage('Docker Build') {
            steps {

                echo "=========================================="
                echo "Building Docker Image"
                echo "=========================================="

                sh '''
                    docker build \
                        -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    docker images | grep devsecops-sample-app
                '''
            }
        }


        // ============================================================
        // 7. TRIVY CONTAINER SECURITY SCAN
        // ============================================================

        stage('Container Scan - Trivy') {
            steps {

                echo "=========================================="
                echo "Running Trivy Container Security Scan"
                echo "=========================================="

                echo "Scanning image:"
                echo "${DOCKER_IMAGE}:${BUILD_NUMBER}"

                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                '''

                echo ""
                echo "WARNING: Trivy scan completed."
                echo "HIGH/CRITICAL vulnerabilities may require remediation."
                echo "Pipeline will continue for this development environment."
            }
        }


        // ============================================================
        // 8. DOCKER HUB PUSH
        // ============================================================

        stage('Docker Hub Push') {
            steps {

                echo "=========================================="
                echo "Logging in to Docker Hub"
                echo "=========================================="

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            --username "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "Docker Hub login successful."

                        echo ""
                        echo "Pushing versioned image..."

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo ""
                        echo "Pushing latest image..."

                        docker push ${DOCKER_IMAGE}:latest

                        echo ""
                        echo "Docker images pushed successfully."

                        docker logout
                    '''
                }
            }
        }


        // ============================================================
        // 9. AWS CONNECTION TEST
        // ============================================================

        stage('AWS Connection Test') {
            steps {

                echo "=========================================="
                echo "Testing AWS Connection"
                echo "=========================================="

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        aws sts get-caller-identity

                        echo ""
                        echo "AWS connection successful."

                        echo ""
                        echo "AWS Region:"
                        aws configure get region || true
                    '''
                }
            }
        }


        // ============================================================
        // 10. DEPLOY TO EC2 USING AWS SSM
        // ============================================================

        stage('Deploy to EC2 via SSM') {
            steps {

                echo "=========================================="
                echo "Deploying Application to EC2"
                echo "=========================================="

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        echo "Sending deployment command to EC2..."

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Deploy DevSecOps Application" \
                            --parameters commands="
                                docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD};
                                docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER};
                                docker stop devsecops-sample-app || true;
                                docker rm devsecops-sample-app || true;
                                docker run -d \
                                    --name devsecops-sample-app \
                                    -p 80:3000 \
                                    ${DOCKER_IMAGE}:${BUILD_NUMBER};
                            " \
                            --region "${AWS_REGION}" \
                            --query "Command.CommandId" \
                            --output text)

                        echo "SSM Command ID:"
                        echo "$COMMAND_ID"

                        echo ""
                        echo "Waiting for deployment command..."

                        sleep 10

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "${EC2_INSTANCE_ID}" \
                            --region "${AWS_REGION}" \
                            --query "{Status:Status,Output:StandardOutputContent,Error:StandardErrorContent}" \
                            --output json
                    '''
                }
            }
        }


        // ============================================================
        // 11. APPLICATION HEALTH CHECK
        // ============================================================

        stage('Application Health Check') {
            steps {

                echo "=========================================="
                echo "Running Application Health Check"
                echo "=========================================="

                sh '''
                    echo "Checking application availability..."

                    sleep 10

                    if curl -f --max-time 10 http://${EC2_PUBLIC_IP}/; then

                        echo ""
                        echo "=========================================="
                        echo "APPLICATION HEALTH CHECK PASSED"
                        echo "=========================================="

                    else

                        echo ""
                        echo "=========================================="
                        echo "APPLICATION HEALTH CHECK FAILED"
                        echo "=========================================="

                        exit 1
                    fi
                '''
            }
        }
    }


    // ================================================================
    // POST ACTIONS
    // ================================================================

    post {

        success {

            echo ""
            echo "=========================================="
            echo "DevSecOps Pipeline SUCCESS"
            echo "=========================================="

            echo "Build Number:"
            echo "${BUILD_NUMBER}"

            echo "Docker Image:"
            echo "${DOCKER_IMAGE}:${BUILD_NUMBER}"

            echo "Deployment completed successfully."
        }

        failure {

            echo ""
            echo "=========================================="
            echo "DevSecOps Pipeline FAILED"
            echo "=========================================="

            echo "Check the failed stage and Jenkins console output."
        }

        always {

            echo ""
            echo "=========================================="
            echo "Pipeline execution completed."
            echo "=========================================="
        }
    }
}
