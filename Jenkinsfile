pipeline {

    agent any

    environment {

        // ============================================================
        // AWS CONFIGURATION
        // ============================================================

        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '042775549160'

        AWS_CREDENTIALS_ID = 'aws-jenkins'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        ECR_REPOSITORY = 'devsecops-sample-app'

        ECR_URI = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        EC2_INSTANCE_ID = 'i-0e0aebd33e4c8e9a1'


        // ============================================================
        // SONARQUBE
        // ============================================================

        SONARQUBE_SERVER = 'sonarqube-dev-sec'


        // ============================================================
        // APPLICATION CONFIGURATION
        // ============================================================

        APP_NAME = 'devsecops-app'

        APP_PORT = '3000'

        HOST_PORT = '80'
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
                    echo "Verifying Application Source"
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
                    echo "All required files found."
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

                withSonarQubeEnv("${SONARQUBE_SERVER}") {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "SonarQube Connection Test"
                        echo "=========================================="

                        echo "SonarQube URL:"
                        echo "${SONAR_HOST_URL}"

                        if [ -n "${SONAR_AUTH_TOKEN}" ]; then
                            echo "Sonar authentication token: CONFIGURED"
                        else
                            echo "Sonar authentication token: NOT CONFIGURED"
                            exit 1
                        fi

                        echo ""
                        echo "SonarQube connection configuration successful."
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
                            echo "Running SonarQube SAST Analysis"
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
                    echo "Installing Application Dependencies"
                    echo "=========================================="

                    npm ci

                    echo ""
                    echo "Dependencies installed successfully."
                '''
            }
        }


        // ============================================================
        // 7. NPM DEPENDENCY AUDIT
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
                    echo "Review dependency vulnerabilities."
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
                    echo "Running Application Test"
                    echo "=========================================="

                    node --check app.js

                    echo ""
                    echo "Application syntax check successful."
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
                    echo "Building Docker Image"
                    echo "=========================================="

                    echo ""
                    echo "Build Number:"
                    echo "${BUILD_NUMBER}"

                    echo ""
                    echo "Image:"
                    echo "${ECR_URI}:${BUILD_NUMBER}"

                    docker build \
                        -t "${ECR_URI}:${BUILD_NUMBER}" \
                        -t "${ECR_URI}:latest" \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    echo ""
                    echo "Docker images:"
                    docker images | grep devsecops-sample-app || true
                '''
            }
        }


        // ============================================================
        // 10. TRIVY CONTAINER SECURITY SCAN
        // ============================================================

        stage('Container Scan - Trivy') {

            steps {

                sh '''
                    echo "=========================================="
                    echo "Running Trivy Container Security Scan"
                    echo "=========================================="

                    echo ""
                    echo "Scanning:"
                    echo "${ECR_URI}:${BUILD_NUMBER}"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        "${ECR_URI}:${BUILD_NUMBER}" || true

                    echo ""
                    echo "WARNING: Trivy scan completed."
                    echo "HIGH/CRITICAL vulnerabilities may require remediation."
                    echo "Pipeline will continue for this development environment."
                '''
            }
        }


        // ============================================================
        // 11. PUSH IMAGE TO ECR
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
                        echo "Pushing Docker Image to Amazon ECR"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo ""
                        echo "AWS Account:"
                        echo "${AWS_ACCOUNT_ID}"

                        echo ""
                        echo "AWS Region:"
                        echo "${AWS_REGION}"

                        echo ""
                        echo "ECR Repository:"
                        echo "${ECR_URI}"

                        echo ""
                        echo "Testing AWS credentials..."

                        aws sts get-caller-identity

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
                        echo "Pushing build image..."

                        docker push \
                            "${ECR_URI}:${BUILD_NUMBER}"

                        echo ""
                        echo "Pushing latest image..."

                        docker push \
                            "${ECR_URI}:latest"

                        echo ""
                        echo "=========================================="
                        echo "ECR PUSH COMPLETED SUCCESSFULLY"
                        echo "=========================================="
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
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "Verifying Image in Amazon ECR"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo ""
                        echo "Repository:"
                        echo "${ECR_REPOSITORY}"

                        echo ""
                        echo "Images currently stored in ECR:"

                        aws ecr list-images \
                            --repository-name "${ECR_REPOSITORY}" \
                            --region "${AWS_REGION}" \
                            --output table

                        echo ""
                        echo "ECR image verification completed successfully."
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
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

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
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "Checking EC2 SSM Connectivity"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=${EC2_INSTANCE_ID}" \
                            --region "${AWS_REGION}" \
                            --output table

                        PING_STATUS=$(aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=${EC2_INSTANCE_ID}" \
                            --region "${AWS_REGION}" \
                            --query 'InstanceInformationList[0].PingStatus' \
                            --output text)

                        echo ""
                        echo "SSM Ping Status:"
                        echo "${PING_STATUS}"

                        if [ "${PING_STATUS}" != "Online" ]; then
                            echo ""
                            echo "ERROR: EC2 instance is not Online in SSM."
                            exit 1
                        fi

                        echo ""
                        echo "EC2 SSM connectivity verified."
                    '''
                }
            }
        }


        // ============================================================
        // 15. DEPLOY TO EC2 VIA SSM
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
                        echo "Deploying Application to EC2"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo ""
                        echo "EC2 Instance:"
                        echo "${EC2_INSTANCE_ID}"

                        echo ""
                        echo "ECR Image:"
                        echo "${ECR_URI}:${BUILD_NUMBER}"


                        # ==================================================
                        # Create SSM command JSON
                        # ==================================================

                        cat > /tmp/ec2-deploy-commands.json <<EOF
{
    "commands": [
        "set -e",
        "echo '=========================================='",
        "echo 'Starting DevSecOps application deployment'",
        "echo '=========================================='",
        "echo 'Logging in to Amazon ECR...'",
        "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}",
        "echo 'ECR login successful.'",
        "echo 'Pulling application image...'",
        "docker pull ${ECR_URI}:${BUILD_NUMBER}",
        "echo 'Stopping previous container...'",
        "docker stop ${APP_NAME} || true",
        "echo 'Removing previous container...'",
        "docker rm ${APP_NAME} || true",
        "echo 'Starting new application container...'",
        "docker run -d --name ${APP_NAME} --restart unless-stopped -p ${HOST_PORT}:${APP_PORT} ${ECR_URI}:${BUILD_NUMBER}",
        "echo 'Waiting for application startup...'",
        "sleep 5",
        "echo 'Checking running containers...'",
        "docker ps",
        "echo 'Checking application container status...'",
        "docker inspect --format '{{.State.Status}}' ${APP_NAME}",
        "echo 'Testing application from EC2...'",
        "curl -f http://localhost:${HOST_PORT}/",
        "echo 'Application health check successful.'",
        "echo '=========================================='",
        "echo 'EC2 deployment completed successfully.'",
        "echo '=========================================='"
    ]
}
EOF


                        echo ""
                        echo "SSM deployment JSON:"
                        cat /tmp/ec2-deploy-commands.json


                        # ==================================================
                        # Send SSM command
                        # ==================================================

                        echo ""
                        echo "Sending deployment command to EC2..."

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "DevSecOps deployment - Jenkins build ${BUILD_NUMBER}" \
                            --parameters file:///tmp/ec2-deploy-commands.json \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
                        echo "SSM Command ID:"
                        echo "${COMMAND_ID}"


                        if [ -z "${COMMAND_ID}" ] || [ "${COMMAND_ID}" = "None" ]; then
                            echo ""
                            echo "ERROR: SSM command was not created."
                            exit 1
                        fi


                        # ==================================================
                        # Wait for deployment
                        # ==================================================

                        echo ""
                        echo "Waiting for EC2 deployment..."

                        DEPLOYMENT_COMPLETE="false"

                        for i in $(seq 1 36); do

                            STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text 2>/dev/null || echo "Pending")

                            echo "Attempt ${i}/36 - Status: ${STATUS}"

                            case "${STATUS}" in

                                Success)

                                    echo ""
                                    echo "=========================================="
                                    echo "EC2 DEPLOYMENT SUCCESSFUL"
                                    echo "=========================================="

                                    echo ""
                                    echo "Deployment Output:"
                                    echo "------------------------------------------"

                                    aws ssm get-command-invocation \
                                        --command-id "${COMMAND_ID}" \
                                        --instance-id "${EC2_INSTANCE_ID}" \
                                        --region "${AWS_REGION}" \
                                        --query 'StandardOutputContent' \
                                        --output text

                                    DEPLOYMENT_COMPLETE="true"

                                    break
                                    ;;


                                Failed|Cancelled|TimedOut|Cancelling)

                                    echo ""
                                    echo "=========================================="
                                    echo "EC2 DEPLOYMENT FAILED"
                                    echo "=========================================="

                                    echo ""
                                    echo "Standard Output:"
                                    echo "------------------------------------------"

                                    aws ssm get-command-invocation \
                                        --command-id "${COMMAND_ID}" \
                                        --instance-id "${EC2_INSTANCE_ID}" \
                                        --region "${AWS_REGION}" \
                                        --query 'StandardOutputContent' \
                                        --output text || true

                                    echo ""
                                    echo "Standard Error:"
                                    echo "------------------------------------------"

                                    aws ssm get-command-invocation \
                                        --command-id "${COMMAND_ID}" \
                                        --instance-id "${EC2_INSTANCE_ID}" \
                                        --region "${AWS_REGION}" \
                                        --query 'StandardErrorContent' \
                                        --output text || true

                                    rm -f /tmp/ec2-deploy-commands.json

                                    exit 1
                                    ;;


                                *)

                                    sleep 5
                                    ;;

                            esac

                        done


                        rm -f /tmp/ec2-deploy-commands.json


                        if [ "${DEPLOYMENT_COMPLETE}" != "true" ]; then

                            echo ""
                            echo "=========================================="
                            echo "EC2 DEPLOYMENT TIMEOUT"
                            echo "=========================================="

                            exit 1

                        fi
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
                        echo "Application Health Check"
                        echo "=========================================="


                        cat > /tmp/health-check.json <<EOF
{
    "commands": [
        "echo 'Checking Docker container...'",
        "docker ps --filter name=${APP_NAME}",
        "echo 'Checking application endpoint...'",
        "curl -f http://localhost:${HOST_PORT}/",
        "echo 'Application is healthy.'"
    ]
}
EOF


                        echo ""
                        echo "Sending health check to EC2..."


                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Application health check - Jenkins build ${BUILD_NUMBER}" \
                            --parameters file:///tmp/health-check.json \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)


                        echo ""
                        echo "Health Check Command ID:"
                        echo "${COMMAND_ID}"


                        HEALTH_CHECK_COMPLETE="false"


                        for i in $(seq 1 24); do

                            STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text 2>/dev/null || echo "Pending")


                            echo "Health check attempt ${i}/24 - Status: ${STATUS}"


                            case "${STATUS}" in

                                Success)

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

                                    HEALTH_CHECK_COMPLETE="true"

                                    break
                                    ;;


                                Failed|Cancelled|TimedOut|Cancelling)

                                    echo ""
                                    echo "=========================================="
                                    echo "APPLICATION HEALTH CHECK FAILED"
                                    echo "=========================================="

                                    echo ""
                                    echo "Output:"
                                    echo "------------------------------------------"

                                    aws ssm get-command-invocation \
                                        --command-id "${COMMAND_ID}" \
                                        --instance-id "${EC2_INSTANCE_ID}" \
                                        --region "${AWS_REGION}" \
                                        --query 'StandardOutputContent' \
                                        --output text || true

                                    echo ""
                                    echo "Error:"
                                    echo "------------------------------------------"

                                    aws ssm get-command-invocation \
                                        --command-id "${COMMAND_ID}" \
                                        --instance-id "${EC2_INSTANCE_ID}" \
                                        --region "${AWS_REGION}" \
                                        --query 'StandardErrorContent' \
                                        --output text || true

                                    rm -f /tmp/health-check.json

                                    exit 1
                                    ;;


                                *)

                                    sleep 5
                                    ;;

                            esac

                        done


                        rm -f /tmp/health-check.json


                        if [ "${HEALTH_CHECK_COMPLETE}" != "true" ]; then

                            echo ""
                            echo "=========================================="
                            echo "APPLICATION HEALTH CHECK TIMEOUT"
                            echo "=========================================="

                            exit 1

                        fi
                    '''
                }
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

Pipeline completed successfully.

Flow:

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
  +-- Trivy
  |
  +-- Amazon ECR
  |
  +-- AWS SSM
  |
  +-- EC2
  |
  +-- Docker Container
  |
  +-- Application Health Check

Application:
http://18.209.29.219

ECR:
042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app
'''
        }


        failure {

            echo '''
==========================================
DEVSECOPS PIPELINE FAILED
==========================================

Check the Jenkins Console Output.

Find the FIRST stage that failed.
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
