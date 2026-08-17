pipeline {

    agent any

    environment {
        // ============================================================
        // AWS / EC2
        // ============================================================
        AWS_REGION       = 'us-east-1'
        EC2_INSTANCE_ID  = 'i-0e0aebd33e4c8e9a1'
        EC2_PUBLIC_IP    = '18.209.29.219'

        // ============================================================
        // ECR
        // ============================================================
        ECR_REGISTRY     = '042775549160.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY   = 'devsecops-sample-app'
        ECR_URI          = '042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app'

        // ============================================================
        // Jenkins Credentials
        // ============================================================
        AWS_CREDENTIALS_ID = 'aws-hrms-v2-taff'

        // ============================================================
        // Docker image
        // ============================================================
        LOCAL_IMAGE = 'devsecops-sample-app'
    }

    stages {

        // ============================================================
        // 1. CHECKOUT
        // ============================================================

        stage('Checkout') {
            steps {
                echo '=========================================='
                echo 'Checking out source code'
                echo '=========================================='

                checkout scm

                echo 'Source code checkout completed successfully.'
            }
        }


        // ============================================================
        // 2. VERIFY SOURCE
        // ============================================================

        stage('Verify Source') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Verifying Application Source"
                    echo "=========================================="

                    ls -la

                    test -f app.js
                    test -f package.json
                    test -f package-lock.json
                    test -f Dockerfile
                    test -f sonar-project.properties

                    echo ""
                    echo "Required files found."
                    echo "Source verification successful."
                '''
            }
        }


        // ============================================================
        // 3. GITLEAKS
        // ============================================================

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Gitleaks Secret Scan"
                    echo "=========================================="

                    gitleaks detect \
                        --source . \
                        --no-banner

                    echo ""
                    echo "Gitleaks scan completed successfully."
                '''
            }
        }


        // ============================================================
        // 4. SONARQUBE CONNECTION TEST
        // ============================================================

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


        // ============================================================
        // 5. SONARQUBE SAST
        // ============================================================

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

                            echo ""
                            echo "SonarQube analysis completed successfully."
                        '''
                    }
                }
            }
        }


        // ============================================================
        // 6. INSTALL DEPENDENCIES
        // ============================================================

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Installing Application Dependencies"
                    echo "=========================================="

                    npm ci

                    echo ""
                    echo "Dependencies installed successfully."
                '''
            }
        }


        // ============================================================
        // 7. NPM AUDIT
        // ============================================================

        stage('Dependency Audit') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running NPM Dependency Audit"
                    echo "=========================================="

                    npm audit --audit-level=high || true

                    echo ""
                    echo "WARNING: NPM audit completed."
                    echo "Vulnerabilities will be reviewed separately."
                    echo "Pipeline will continue for this development environment."
                '''
            }
        }


        // ============================================================
        // 8. APPLICATION TEST
        // ============================================================

        stage('Application Test') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Application Test"
                    echo "=========================================="

                    node --check app.js

                    echo ""
                    echo "Node.js syntax check successful."
                '''
            }
        }


        // ============================================================
        // 9. DOCKER BUILD
        // ============================================================

        stage('Docker Build') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Building Docker Image"
                    echo "=========================================="

                    echo "Image:"
                    echo "${LOCAL_IMAGE}:${BUILD_NUMBER}"

                    docker build \
                        -t "${LOCAL_IMAGE}:${BUILD_NUMBER}" \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    docker images | grep devsecops-sample-app
                '''
            }
        }


        // ============================================================
        // 10. TRIVY
        // ============================================================

        stage('Container Scan - Trivy') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Trivy Container Security Scan"
                    echo "=========================================="

                    echo "Scanning:"
                    echo "${LOCAL_IMAGE}:${BUILD_NUMBER}"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        "${LOCAL_IMAGE}:${BUILD_NUMBER}" || true

                    echo ""
                    echo "WARNING: Trivy scan completed."
                    echo "HIGH/CRITICAL vulnerabilities may require remediation."
                    echo "Pipeline will continue for this development environment."
                '''
            }
        }


        // ============================================================
        // 11. PUSH TO ECR
        // ============================================================

        stage('Push Image to ECR') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        echo "=========================================="
                        echo "Pushing Docker Image to Amazon ECR"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo "AWS Region:"
                        echo "${AWS_REGION}"

                        echo ""
                        echo "ECR Repository:"
                        echo "${ECR_URI}"

                        echo ""
                        echo "Logging in to Amazon ECR..."

                        aws ecr get-login-password \
                            --region "${AWS_REGION}" | \
                        docker login \
                            --username AWS \
                            --password-stdin "${ECR_REGISTRY}"

                        echo ""
                        echo "ECR login successful."

                        echo ""
                        echo "Tagging build image..."

                        docker tag \
                            "${LOCAL_IMAGE}:${BUILD_NUMBER}" \
                            "${ECR_URI}:${BUILD_NUMBER}"

                        docker tag \
                            "${LOCAL_IMAGE}:${BUILD_NUMBER}" \
                            "${ECR_URI}:latest"

                        echo ""
                        echo "Images to push:"
                        echo "${ECR_URI}:${BUILD_NUMBER}"
                        echo "${ECR_URI}:latest"

                        echo ""
                        echo "Pushing build image..."

                        docker push \
                            "${ECR_URI}:${BUILD_NUMBER}"

                        echo ""
                        echo "Pushing latest image..."

                        docker push \
                            "${ECR_URI}:latest"

                        echo ""
                        echo "ECR push completed successfully."
                    '''
                }
            }
        }


        // ============================================================
        // 12. VERIFY ECR IMAGE
        // ============================================================

        stage('Verify ECR Image') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        echo "=========================================="
                        echo "Verifying Image in Amazon ECR"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws ecr describe-images \
                            --repository-name "${ECR_REPOSITORY}" \
                            --region "${AWS_REGION}" \
                            --query 'imageDetails[*].imageTags[]' \
                            --output table

                        echo ""
                        echo "ECR image verification completed."
                    '''
                }
            }
        }


        // ============================================================
        // 13. AWS CONNECTION TEST
        // ============================================================

        stage('AWS Connection Test') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        echo "=========================================="
                        echo "Testing AWS Connection"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws sts get-caller-identity

                        echo ""
                        echo "AWS connection successful."
                    '''
                }
            }
        }


        // ============================================================
        // 14. VERIFY EC2 SSM CONNECTION
        // ============================================================

        stage('Verify EC2 SSM Connection') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        echo "=========================================="
                        echo "Checking EC2 SSM Connectivity"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=${EC2_INSTANCE_ID}" \
                            --region "${AWS_REGION}" \
                            --output table

                        echo ""
                        echo "EC2 SSM connectivity verified."
                    '''
                }
            }
        }


        // ============================================================
        // 15. DEPLOY TO EC2 USING SSM
        // ============================================================

        stage('Deploy to EC2 via SSM') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        echo "=========================================="
                        echo "Deploying Application to EC2"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo "EC2 Instance:"
                        echo "${EC2_INSTANCE_ID}"

                        echo ""
                        echo "ECR Image:"
                        echo "${ECR_URI}:${BUILD_NUMBER}"

                        echo ""
                        echo "Sending deployment command to EC2..."

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Deploy DevSecOps application - Jenkins build ${BUILD_NUMBER}" \
                            --parameters commands="
                                set -e
                                echo 'Starting EC2 deployment...'
                                echo 'Logging in to ECR...'
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                echo 'Pulling application image...'
                                docker pull ${ECR_URI}:${BUILD_NUMBER}
                                echo 'Stopping old container...'
                                docker stop devsecops-app || true
                                echo 'Removing old container...'
                                docker rm devsecops-app || true
                                echo 'Starting new container...'
                                docker run -d --name devsecops-app --restart unless-stopped -p 80:3000 ${ECR_URI}:${BUILD_NUMBER}
                                echo 'Checking running containers...'
                                docker ps
                                echo 'Checking application container...'
                                docker inspect --format '{{.State.Status}}' devsecops-app
                                echo 'Deployment completed.'
                            " \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
                        echo "SSM Command ID:"
                        echo "${COMMAND_ID}"

                        echo ""
                        echo "Waiting for deployment to complete..."

                        DEPLOYMENT_STATUS="Pending"

                        for i in $(seq 1 36); do

                            DEPLOYMENT_STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text)

                            echo "Attempt ${i}/36 - Status: ${DEPLOYMENT_STATUS}"

                            if [ "${DEPLOYMENT_STATUS}" = "Success" ]; then
                                break
                            fi

                            if [ "${DEPLOYMENT_STATUS}" = "Failed" ] || \
                               [ "${DEPLOYMENT_STATUS}" = "Cancelled" ] || \
                               [ "${DEPLOYMENT_STATUS}" = "TimedOut" ] || \
                               [ "${DEPLOYMENT_STATUS}" = "Cancelling" ]; then

                                echo ""
                                echo "=========================================="
                                echo "EC2 DEPLOYMENT FAILED"
                                echo "=========================================="

                                echo ""
                                echo "Standard Output:"
                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardOutputContent' \
                                    --output text

                                echo ""
                                echo "Standard Error:"
                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardErrorContent' \
                                    --output text

                                exit 1
                            fi

                            sleep 5
                        done

                        echo ""
                        echo "Final deployment status:"
                        echo "${DEPLOYMENT_STATUS}"

                        echo ""
                        echo "Deployment output:"

                        aws ssm get-command-invocation \
                            --command-id "${COMMAND_ID}" \
                            --instance-id "${EC2_INSTANCE_ID}" \
                            --region "${AWS_REGION}" \
                            --query '[StandardOutputContent,StandardErrorContent]' \
                            --output text

                        if [ "${DEPLOYMENT_STATUS}" != "Success" ]; then
                            echo ""
                            echo "Deployment timed out."
                            exit 1
                        fi

                        echo ""
                        echo "EC2 deployment completed successfully."
                    '''
                }
            }
        }


        // ============================================================
        // 16. APPLICATION HEALTH CHECK
        // ============================================================

        stage('Application Health Check') {
            steps {

                sh '''
                    echo "=========================================="
                    echo "Application Health Check"
                    echo "=========================================="

                    echo "Application URL:"
                    echo "http://${EC2_PUBLIC_IP}"

                    echo ""
                    echo "Waiting for application startup..."

                    sleep 10

                    echo ""
                    echo "Checking HTTP endpoint..."

                    HTTP_STATUS=$(curl \
                        --silent \
                        --output /tmp/devsecops-response.txt \
                        --write-out "%{http_code}" \
                        --max-time 15 \
                        "http://${EC2_PUBLIC_IP}/" || true)

                    echo ""
                    echo "HTTP Status:"
                    echo "${HTTP_STATUS}"

                    echo ""
                    echo "Application Response:"
                    cat /tmp/devsecops-response.txt || true

                    if [ "${HTTP_STATUS}" = "200" ]; then

                        echo ""
                        echo "=========================================="
                        echo "APPLICATION HEALTH CHECK PASSED"
                        echo "=========================================="

                    else

                        echo ""
                        echo "=========================================="
                        echo "APPLICATION HEALTH CHECK FAILED"
                        echo "=========================================="

                        echo "Expected HTTP 200."
                        echo "Received HTTP ${HTTP_STATUS}."

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
            echo '''
==========================================
DEVSECOPS PIPELINE SUCCESS
==========================================

Build completed successfully.

Application deployed to EC2.

Application URL:
http://18.209.29.219

Docker image:
042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app:${BUILD_NUMBER}
'''
        }

        failure {
            echo '''
==========================================
DEVSECOPS PIPELINE FAILED
==========================================

Check the first failed stage in the Jenkins console output.
'''
        }

        always {
            echo '''
==========================================
Pipeline execution completed.
==========================================
'''
        }
    }
}
