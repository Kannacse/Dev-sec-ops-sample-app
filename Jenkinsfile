pipeline {

    agent any

    environment {

        // ============================================================
        // AWS CONFIGURATION
        // ============================================================

        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '042775549160'

        // Jenkins credential ID
        AWS_CREDENTIALS_ID = 'aws-jenkins'

        // ECR
        ECR_REGISTRY = '042775549160.dkr.ecr.us-east-1.amazonaws.com'

        ECR_REPOSITORY = 'devsecops-sample-app'

        ECR_URI = '042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app'

        // EC2
        EC2_INSTANCE_ID = 'i-0e0aebd33e4c8e9a1'

        // ============================================================
        // SONARQUBE
        // ============================================================

        SONARQUBE_SERVER = 'sonarqube-dev-sec'

        // ============================================================
        // APPLICATION
        // ============================================================

        APP_NAME = 'devsecops-sample-app'

        APP_PORT = '3000'

        HOST_PORT = '3000'
    }


    stages {


        // ============================================================
        // 1. CHECKOUT
        // ============================================================

        stage('Checkout') {

            steps {

                checkout scm

            }
        }


        // ============================================================
        // 2. VERIFY SOURCE
        // ============================================================

        stage('Verify Source') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "VERIFYING APPLICATION SOURCE"
                    echo "=========================================="

                    echo ""
                    echo "Workspace:"
                    pwd

                    echo ""
                    echo "Files:"
                    ls -la

                    echo ""
                    echo "Checking required files..."

                    test -f app.js
                    test -f package.json
                    test -f package-lock.json
                    test -f Dockerfile
                    test -f Jenkinsfile
                    test -f sonar-project.properties

                    echo ""
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
                    set -e

                    echo "=========================================="
                    echo "GITLEAKS SECRET SCAN"
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
        // 4. SONARQUBE CONNECTION
        // ============================================================

        stage('SonarQube Connection Test') {

            steps {

                withSonarQubeEnv("${SONARQUBE_SERVER}") {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "SONARQUBE CONNECTION TEST"
                        echo "=========================================="

                        echo "SonarQube URL:"
                        echo "${SONAR_HOST_URL}"

                        if [ -z "${SONAR_AUTH_TOKEN}" ]; then
                            echo "ERROR: SonarQube token is not configured."
                            exit 1
                        fi

                        echo ""
                        echo "SonarQube authentication token: CONFIGURED"
                    '''
                }
            }
        }


        // ============================================================
        // 5. SONARQUBE SAST
        // ============================================================

        stage('SAST - SonarQube') {

            steps {

                withSonarQubeEnv("${SONARQUBE_SERVER}") {

                    withEnv([
                        "PATH+SONAR=/opt/sonar-scanner/bin"
                    ]) {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "SONARQUBE SAST ANALYSIS"
                            echo "=========================================="

                            sonar-scanner \
                                -Dsonar.token="${SONAR_AUTH_TOKEN}"

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
                    set -e

                    echo "=========================================="
                    echo "INSTALLING DEPENDENCIES"
                    echo "=========================================="

                    npm ci

                    echo ""
                    echo "Dependencies installed successfully."
                '''
            }
        }


        // ============================================================
        // 7. NPM AUDIT
        // WARNING ONLY
        // ============================================================

        stage('Dependency Audit') {

            steps {

                sh '''
                    echo "=========================================="
                    echo "NPM DEPENDENCY AUDIT"
                    echo "=========================================="

                    npm audit --audit-level=high || true

                    echo ""
                    echo "WARNING: npm audit completed."
                    echo "Pipeline will continue."
                '''
            }
        }


        // ============================================================
        // 8. APPLICATION TEST
        // ============================================================

        stage('Application Test') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "APPLICATION TEST"
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
                    set -e

                    echo "=========================================="
                    echo "DOCKER BUILD"
                    echo "=========================================="

                    echo "Build Number: ${BUILD_NUMBER}"

                    echo "Building:"
                    echo "${ECR_URI}:${BUILD_NUMBER}"

                    docker build \
                        -t "${ECR_URI}:${BUILD_NUMBER}" \
                        -t "${ECR_URI}:latest" \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    docker images | grep devsecops-sample-app || true
                '''
            }
        }


        // ============================================================
        // 10. TRIVY
        // WARNING ONLY
        // ============================================================

        stage('Container Scan - Trivy') {

            steps {

                sh '''
                    echo "=========================================="
                    echo "TRIVY CONTAINER SECURITY SCAN"
                    echo "=========================================="

                    echo ""
                    echo "Scanning:"
                    echo "${ECR_URI}:${BUILD_NUMBER}"

                    echo ""

                    trivy image \
                        --severity HIGH,CRITICAL \
                        "${ECR_URI}:${BUILD_NUMBER}" || true

                    echo ""
                    echo "=========================================="
                    echo "WARNING: TRIVY SCAN COMPLETED"
                    echo "=========================================="

                    echo ""
                    echo "HIGH/CRITICAL vulnerabilities may exist."
                    echo "Pipeline will continue for this development environment."
                '''
            }
        }


        // ============================================================
        // 11. AWS CONNECTION TEST
        // ============================================================

        stage('AWS Connection Test') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "AWS CONNECTION TEST"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws sts get-caller-identity

                        echo ""
                        echo "AWS authentication successful."
                    '''
                }
            }
        }


        // ============================================================
        // 12. ECR LOGIN AND PUSH
        // ============================================================

        stage('Push Image to ECR') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "PUSHING IMAGE TO AMAZON ECR"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo ""
                        echo "ECR Registry:"
                        echo "${ECR_REGISTRY}"

                        echo ""
                        echo "ECR Repository:"
                        echo "${ECR_REPOSITORY}"

                        echo ""
                        echo "Logging into ECR..."

                        aws ecr get-login-password \
                            --region "${AWS_REGION}" | \
                        docker login \
                            --username AWS \
                            --password-stdin \
                            "${ECR_REGISTRY}"

                        echo ""
                        echo "ECR login successful."

                        echo ""
                        echo "Pushing build image..."

                        docker push \
                            "${ECR_URI}:${BUILD_NUMBER}"

                        echo ""
                        echo "Pushing latest image..."

                        docker push \
                            "${ECR_URI}:latest"

                        echo ""
                        echo "=========================================="
                        echo "ECR PUSH SUCCESSFUL"
                        echo "=========================================="
                    '''
                }
            }
        }


        // ============================================================
        // 13. VERIFY ECR
        // ============================================================

        stage('Verify ECR Image') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "VERIFYING ECR IMAGE"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws ecr list-images \
                            --repository-name "${ECR_REPOSITORY}" \
                            --region "${AWS_REGION}" \
                            --output table

                        echo ""
                        echo "ECR verification completed."
                    '''
                }
            }
        }


        // ============================================================
        // 14. VERIFY EC2 SSM
        // ============================================================

        stage('Verify EC2 SSM Connection') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "VERIFYING EC2 SSM CONNECTION"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        PING_STATUS=$(aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=${EC2_INSTANCE_ID}" \
                            --region "${AWS_REGION}" \
                            --query 'InstanceInformationList[0].PingStatus' \
                            --output text)

                        echo ""
                        echo "EC2 Instance:"
                        echo "${EC2_INSTANCE_ID}"

                        echo ""
                        echo "SSM Status:"
                        echo "${PING_STATUS}"

                        if [ "${PING_STATUS}" != "Online" ]; then

                            echo ""
                            echo "ERROR: EC2 is not Online in SSM."
                            exit 1

                        fi

                        echo ""
                        echo "EC2 SSM connection successful."
                    '''
                }
            }
        }


        // ============================================================
        // 15. DEPLOY TO EC2 THROUGH SSM
        // ============================================================

        stage('Deploy to EC2 via SSM') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "DEPLOYING APPLICATION TO EC2"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo ""
                        echo "EC2:"
                        echo "${EC2_INSTANCE_ID}"

                        echo ""
                        echo "Image:"
                        echo "${ECR_URI}:${BUILD_NUMBER}"


                        # ------------------------------------------------
                        # Create SSM command document
                        # ------------------------------------------------

                        cat > /tmp/deploy-commands.json <<EOF
{
    "commands": [
        "set -e",
        "echo '=========================================='",
        "echo 'EC2 DEPLOYMENT STARTED'",
        "echo '=========================================='",
        "echo 'Checking AWS CLI...'",
        "aws --version",
        "echo 'Checking Docker...'",
        "docker --version",
        "echo 'Logging into Amazon ECR...'",
        "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}",
        "echo 'ECR login successful.'",
        "echo 'Pulling image from ECR...'",
        "docker pull ${ECR_URI}:${BUILD_NUMBER}",
        "echo 'Stopping old application container...'",
        "docker stop ${APP_NAME} || true",
        "echo 'Removing old application container...'",
        "docker rm ${APP_NAME} || true",
        "echo 'Starting new application container...'",
        "docker run -d --name ${APP_NAME} --restart unless-stopped -p ${HOST_PORT}:${APP_PORT} ${ECR_URI}:${BUILD_NUMBER}",
        "echo 'Waiting for application...'",
        "sleep 5",
        "echo 'Running containers:'",
        "docker ps",
        "echo 'Testing application locally...'",
        "curl -f http://localhost:${HOST_PORT}/",
        "echo '=========================================='",
        "echo 'EC2 DEPLOYMENT SUCCESSFUL'",
        "echo '=========================================='"
    ]
}
EOF


                        echo ""
                        echo "Sending deployment command to EC2..."


                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "DevSecOps deployment build ${BUILD_NUMBER}" \
                            --parameters file:///tmp/deploy-commands.json \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)


                        echo ""
                        echo "SSM Command ID:"
                        echo "${COMMAND_ID}"


                        if [ -z "${COMMAND_ID}" ] || [ "${COMMAND_ID}" = "None" ]; then
                            echo "ERROR: Failed to create SSM command."
                            exit 1
                        fi


                        # ------------------------------------------------
                        # Wait for deployment
                        # ------------------------------------------------

                        echo ""
                        echo "Waiting for EC2 deployment..."


                        DEPLOYMENT_STATUS="Pending"


                        for i in $(seq 1 36); do

                            DEPLOYMENT_STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text 2>/dev/null || echo "Pending")


                            echo "Attempt ${i}/36"
                            echo "Status: ${DEPLOYMENT_STATUS}"


                            if [ "${DEPLOYMENT_STATUS}" = "Success" ]; then

                                echo ""
                                echo "=========================================="
                                echo "EC2 DEPLOYMENT SUCCESSFUL"
                                echo "=========================================="

                                echo ""
                                echo "Output:"
                                echo "------------------------------------------"

                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardOutputContent' \
                                    --output text

                                break

                            fi


                            if [ "${DEPLOYMENT_STATUS}" = "Failed" ] || \
                               [ "${DEPLOYMENT_STATUS}" = "Cancelled" ] || \
                               [ "${DEPLOYMENT_STATUS}" = "TimedOut" ]; then

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
                                    --output text || true

                                echo ""
                                echo "Standard Error:"
                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardErrorContent' \
                                    --output text || true

                                exit 1

                            fi


                            sleep 5

                        done


                        if [ "${DEPLOYMENT_STATUS}" != "Success" ]; then

                            echo ""
                            echo "ERROR: Deployment timed out."

                            exit 1

                        fi


                        rm -f /tmp/deploy-commands.json
                    '''
                }
            }
        }


        // ============================================================
        // 16. APPLICATION HEALTH CHECK
        // ============================================================

        stage('Application Health Check') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "APPLICATION HEALTH CHECK"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"


                        cat > /tmp/health-check.json <<EOF
{
    "commands": [
        "echo 'Checking application container...'",
        "docker ps --filter name=${APP_NAME}",
        "echo 'Checking application endpoint...'",
        "curl -f http://localhost:${HOST_PORT}/",
        "echo 'Application health check passed.'"
    ]
}
EOF


                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Health check build ${BUILD_NUMBER}" \
                            --parameters file:///tmp/health-check.json \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)


                        echo ""
                        echo "Health Check Command ID:"
                        echo "${COMMAND_ID}"


                        HEALTH_STATUS="Pending"


                        for i in $(seq 1 24); do

                            HEALTH_STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text 2>/dev/null || echo "Pending")


                            echo "Health check ${i}/24: ${HEALTH_STATUS}"


                            if [ "${HEALTH_STATUS}" = "Success" ]; then

                                echo ""
                                echo "Health Check Output:"
                                echo "------------------------------------------"

                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardOutputContent' \
                                    --output text

                                echo ""
                                echo "=========================================="
                                echo "APPLICATION HEALTH CHECK PASSED"
                                echo "=========================================="

                                break

                            fi


                            if [ "${HEALTH_STATUS}" = "Failed" ] || \
                               [ "${HEALTH_STATUS}" = "Cancelled" ] || \
                               [ "${HEALTH_STATUS}" = "TimedOut" ]; then

                                echo ""
                                echo "=========================================="
                                echo "APPLICATION HEALTH CHECK FAILED"
                                echo "=========================================="

                                echo ""
                                echo "Output:"
                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardOutputContent' \
                                    --output text || true

                                echo ""
                                echo "Error:"
                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardErrorContent' \
                                    --output text || true

                                exit 1

                            fi


                            sleep 5

                        done


                        rm -f /tmp/health-check.json


                        if [ "${HEALTH_STATUS}" != "Success" ]; then

                            echo ""
                            echo "ERROR: Health check timed out."

                            exit 1

                        fi
                    '''
                }
            }
        }
    }


    // ================================================================
    // POST
    // ================================================================

    post {

        success {

            echo '''
==========================================
DEVSECOPS PIPELINE SUCCESS
==========================================

GitHub
   |
   v
Jenkins
   |
   +-- Gitleaks
   |
   +-- SonarQube
   |
   +-- NPM Audit
   |
   +-- Application Test
   |
   +-- Docker Build
   |
   +-- Trivy WARNING
   |
   +-- Amazon ECR
   |
   +-- AWS SSM
   |
   +-- EC2
   |
   +-- Docker Container
   |
   +-- Health Check
   |
   v
APPLICATION DEPLOYED

Expected URL:

http://18.209.29.219:3000
'''
        }


        failure {

            echo '''
==========================================
DEVSECOPS PIPELINE FAILED
==========================================

Check the FIRST failed stage in Console Output.
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
