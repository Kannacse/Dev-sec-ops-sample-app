pipeline {

    agent any

    environment {

        // ============================================================
        // AWS
        // ============================================================

        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '042775549160'

        // IMPORTANT:
        // This must exactly match the Jenkins credential ID.
        AWS_CREDENTIALS_ID = 'aws-hrms-v2-taff'

        // ============================================================
        // ECR
        // ============================================================

        ECR_REGISTRY = '042775549160.dkr.ecr.us-east-1.amazonaws.com'

        ECR_REPOSITORY = 'devsecops-sample-app'

        ECR_URI = '042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app'

        // ============================================================
        // EC2
        // ============================================================

        EC2_INSTANCE_ID = 'i-096fc3c14a9db3ad8'

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

                    echo ""
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
        // 13. GET ECR DIGEST
        // ============================================================

        stage('Get ECR Image Digest') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}",
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {

                    script {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "GETTING ECR IMAGE DIGEST"
                            echo "=========================================="

                            export AWS_DEFAULT_REGION="${AWS_REGION}"

                            aws ecr describe-images \
                                --repository-name "${ECR_REPOSITORY}" \
                                --image-ids imageTag="${BUILD_NUMBER}" \
                                --region "${AWS_REGION}" \
                                --query 'imageDetails[0].imageDigest' \
                                --output text
                        '''

                        env.ECR_IMAGE_DIGEST = sh(
                            script: '''
                                aws ecr describe-images \
                                    --repository-name "${ECR_REPOSITORY}" \
                                    --image-ids imageTag="${BUILD_NUMBER}" \
                                    --region "${AWS_REGION}" \
                                    --query 'imageDetails[0].imageDigest' \
                                    --output text
                            ''',
                            returnStdout: true
                        ).trim()

                        if (!env.ECR_IMAGE_DIGEST ||
                            env.ECR_IMAGE_DIGEST == 'None') {

                            error("ECR image digest was not found.")
                        }

                        echo "ECR Image Digest:"
                        echo "${env.ECR_IMAGE_DIGEST}"
                    }
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
        //
        // IMPORTANT:
        // The deployment script is BASE64 encoded before being sent
        // through SSM.
        //
        // This prevents the long ECR image reference from being
        // accidentally changed/truncated by shell/SSM parsing.
        //
        // The EC2 pulls the image by ECR DIGEST.
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
                        echo "ECR Image:"
                        echo "${ECR_URI}@${ECR_IMAGE_DIGEST}"

                        # ------------------------------------------------
                        # Create deployment script
                        # ------------------------------------------------

                        cat > /tmp/ec2-deploy.sh <<EOF
#!/bin/bash

set -e

echo "=========================================="
echo "EC2 DEPLOYMENT STARTED"
echo "=========================================="

echo ""
echo "Checking AWS CLI..."
aws --version

echo ""
echo "Checking Docker..."
docker --version

echo ""
echo "Checking curl..."
curl --version | head -n 1

echo ""
echo "ECR Registry:"
echo "${ECR_REGISTRY}"

echo ""
echo "Application:"
echo "${APP_NAME}"

echo ""
echo "Image:"
echo "${ECR_URI}@${ECR_IMAGE_DIGEST}"

echo ""
echo "Logging into Amazon ECR..."

aws ecr get-login-password \
    --region "${AWS_REGION}" | \
docker login \
    --username AWS \
    --password-stdin \
    "${ECR_REGISTRY}"

echo ""
echo "ECR login successful."

echo ""
echo "Pulling exact image by digest..."

docker pull \
    "${ECR_URI}@${ECR_IMAGE_DIGEST}"

echo ""
echo "Stopping existing application container..."

docker stop "${APP_NAME}" || true

echo ""
echo "Removing existing application container..."

docker rm "${APP_NAME}" || true

echo ""
echo "Starting new application container..."

docker run -d \
    --name "${APP_NAME}" \
    --restart unless-stopped \
    --publish "${HOST_PORT}:${APP_PORT}" \
    "${ECR_URI}@${ECR_IMAGE_DIGEST}"

echo ""
echo "Waiting for application startup..."

sleep 5

echo ""
echo "Running containers:"

docker ps

echo ""
echo "Checking application endpoint..."

curl -f \
    "http://localhost:${HOST_PORT}/"

echo ""
echo "=========================================="
echo "EC2 DEPLOYMENT SUCCESSFUL"
echo "=========================================="
EOF

                        # ------------------------------------------------
                        # Encode script so SSM cannot mangle it
                        # ------------------------------------------------

                        DEPLOY_SCRIPT_B64=$(base64 -w 0 /tmp/ec2-deploy.sh)

                        echo ""
                        echo "Deployment script encoded successfully."

                        # ------------------------------------------------
                        # Create ONE short SSM command
                        # ------------------------------------------------

                        SSM_COMMAND="echo ${DEPLOY_SCRIPT_B64} | base64 -d > /tmp/ec2-deploy.sh && chmod +x /tmp/ec2-deploy.sh && /tmp/ec2-deploy.sh"

                        # ------------------------------------------------
                        # Send command to EC2
                        # ------------------------------------------------

                        echo ""
                        echo "Sending deployment command to EC2..."

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "DevSecOps deployment build ${BUILD_NUMBER}" \
                            --parameters "commands=[\"${SSM_COMMAND}\"]" \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
                        echo "SSM Command ID:"
                        echo "${COMMAND_ID}"

                        if [ -z "${COMMAND_ID}" ] ||
                           [ "${COMMAND_ID}" = "None" ]; then

                            echo "ERROR: Failed to create SSM command."
                            exit 1

                        fi

                        # ------------------------------------------------
                        # Wait for deployment
                        # ------------------------------------------------

                        echo ""
                        echo "Waiting for EC2 deployment..."

                        DEPLOYMENT_STATUS="Pending"

                        for i in $(seq 1 60); do

                            DEPLOYMENT_STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text 2>/dev/null || echo "Pending")

                            echo "Attempt ${i}/60"
                            echo "Status: ${DEPLOYMENT_STATUS}"

                            if [ "${DEPLOYMENT_STATUS}" = "Success" ]; then

                                echo ""
                                echo "=========================================="
                                echo "EC2 DEPLOYMENT SUCCESSFUL"
                                echo "=========================================="

                                echo ""
                                echo "EC2 Output:"
                                echo "------------------------------------------"

                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --region "${AWS_REGION}" \
                                    --query 'StandardOutputContent' \
                                    --output text

                                break

                            fi

                            if [ "${DEPLOYMENT_STATUS}" = "Failed" ] ||
                               [ "${DEPLOYMENT_STATUS}" = "Cancelled" ] ||
                               [ "${DEPLOYMENT_STATUS}" = "TimedOut" ]; then

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

                                exit 1

                            fi

                            sleep 5

                        done

                        if [ "${DEPLOYMENT_STATUS}" != "Success" ]; then

                            echo ""
                            echo "ERROR: EC2 deployment timed out."

                            exit 1

                        fi

                        rm -f /tmp/ec2-deploy.sh
                    '''
                }
            }
        }


        // ============================================================
        // 16. FINAL APPLICATION HEALTH CHECK
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
                        echo "FINAL APPLICATION HEALTH CHECK"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        cat > /tmp/ec2-health.sh <<EOF
#!/bin/bash

set -e

echo "Checking application container..."

docker ps \
    --filter "name=${APP_NAME}" \
    --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"

echo ""
echo "Checking application endpoint..."

curl -f \
    "http://localhost:${HOST_PORT}/"

echo ""
echo "Application health check passed."
EOF

                        HEALTH_SCRIPT_B64=$(base64 -w 0 /tmp/ec2-health.sh)

                        SSM_HEALTH_COMMAND="echo ${HEALTH_SCRIPT_B64} | base64 -d > /tmp/ec2-health.sh && chmod +x /tmp/ec2-health.sh && /tmp/ec2-health.sh"

                        echo ""
                        echo "Sending health check to EC2..."

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Application health check build ${BUILD_NUMBER}" \
                            --parameters "commands=[\"${SSM_HEALTH_COMMAND}\"]" \
                            --region "${AWS_REGION}" \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
                        echo "Health Check Command ID:"
                        echo "${COMMAND_ID}"

                        HEALTH_STATUS="Pending"

                        for i in $(seq 1 36); do

                            HEALTH_STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --region "${AWS_REGION}" \
                                --query 'Status' \
                                --output text 2>/dev/null || echo "Pending")

                            echo "Health check ${i}/36: ${HEALTH_STATUS}"

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

                            if [ "${HEALTH_STATUS}" = "Failed" ] ||
                               [ "${HEALTH_STATUS}" = "Cancelled" ] ||
                               [ "${HEALTH_STATUS}" = "TimedOut" ]; then

                                echo ""
                                echo "=========================================="
                                echo "APPLICATION HEALTH CHECK FAILED"
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

                        rm -f /tmp/ec2-health.sh

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
    // POST ACTIONS
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
Jenkins - Local Ubuntu
   |
   +-- Gitleaks
   |
   +-- SonarQube SAST
   |
   +-- NPM Audit WARNING
   |
   +-- Application Test
   |
   +-- Docker Build
   |
   +-- Trivy WARNING
   |
   +-- Amazon ECR
   |
   +-- ECR Image Digest
   |
   +-- AWS SSM
   |
   +-- EC2
   |
   +-- Docker Container
   |
   +-- Application Health Check
   |
   v
APPLICATION DEPLOYED

EC2:
i-096fc3c14a9db3ad8

Application:
Port 3000
'''
        }


        failure {

            echo '''
==========================================
DEVSECOPS PIPELINE FAILED
==========================================

Check the FIRST failed stage in Console Output.

Possible areas:

1. Gitleaks
2. SonarQube
3. npm dependencies
4. Application test
5. Docker
6. Trivy
7. AWS credentials
8. ECR
9. SSM
10. EC2 Docker deployment
11. Application health check
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
