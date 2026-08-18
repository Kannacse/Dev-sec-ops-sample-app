pipeline {

    agent any

    environment {

        // ============================================================
        // AWS
        // ============================================================

        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '042775549160'

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
        // APPLICATION
        // ============================================================

        APP_NAME = 'devsecops-sample-app'
        APP_PORT = '3000'
        HOST_PORT = '3000'

        // ============================================================
        // SONARQUBE
        // ============================================================

        SONARQUBE_SERVER = 'sonarqube-dev-sec'
        SONAR_TOKEN_CREDENTIAL_ID = 'sonarqube-jenkins-token'

        // ============================================================
        // SONAR SCANNER
        // ============================================================

        SONAR_SCANNER_HOME = '/opt/sonar-scanner'
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
                    echo "Gitleaks scan passed."
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
                        echo "$SONAR_HOST_URL"

                        curl -fsS \
                            "$SONAR_HOST_URL/api/system/status"

                        echo ""
                        echo "SonarQube connection successful."
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

                    withCredentials([
                        string(
                            credentialsId: "${SONAR_TOKEN_CREDENTIAL_ID}",
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "SONARQUBE SAST SCAN"
                            echo "=========================================="

                            export PATH="${SONAR_SCANNER_HOME}/bin:$PATH"

                            echo ""
                            echo "Checking SonarScanner..."

                            sonar-scanner --version

                            echo ""
                            echo "Running SonarQube SAST..."

                            sonar-scanner \
                                -Dsonar.token="$SONAR_TOKEN"

                            echo ""
                            echo "SonarQube SAST scan completed."
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
                    echo "INSTALLING NODE DEPENDENCIES"
                    echo "=========================================="

                    node --version
                    npm --version

                    npm ci

                    echo ""
                    echo "Dependencies installed successfully."
                '''
            }
        }


        // ============================================================
        // 7. DEPENDENCY AUDIT
        // ============================================================

        stage('Dependency Audit') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "NPM DEPENDENCY SECURITY AUDIT"
                    echo "=========================================="

                    npm audit --audit-level=high

                    echo ""
                    echo "Dependency audit passed."
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

                    npm test --if-present

                    echo ""
                    echo "Application test completed."
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

                    docker --version

                    docker build \
                        --pull \
                        -t "$ECR_URI:$BUILD_NUMBER" \
                        -t "$ECR_URI:latest" \
                        .

                    echo ""
                    echo "Docker build successful."

                    docker images "$ECR_URI"
                '''
            }
        }


        // ============================================================
        // 10. TRIVY CONTAINER SCAN
        // ============================================================

        stage('Container Scan - Trivy') {
            steps {
                sh '''
                    set -e

                    echo "=========================================="
                    echo "TRIVY CONTAINER SECURITY SCAN"
                    echo "=========================================="

                    trivy --version

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        "$ECR_URI:$BUILD_NUMBER"

                    echo ""
                    echo "Trivy scan passed."
                '''
            }
        }


        // ============================================================
        // 11. AWS CONNECTION
        // ============================================================

        stage('AWS Connection Test') {
            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "AWS CONNECTION TEST"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        aws --version

                        echo ""
                        echo "AWS Identity:"

                        aws sts get-caller-identity

                        echo ""
                        echo "AWS connection successful."
                    '''
                }
            }
        }


        // ============================================================
        // 12. PUSH IMAGE TO ECR
        // ============================================================

        stage('Push Image to ECR') {
            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "PUSHING IMAGE TO ECR"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo "Logging into ECR..."

                        aws ecr get-login-password \
                            --region "$AWS_REGION" |
                            docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        echo ""
                        echo "Pushing build-number image..."

                        docker push \
                            "$ECR_URI:$BUILD_NUMBER"

                        echo ""
                        echo "Pushing latest image..."

                        docker push \
                            "$ECR_URI:latest"

                        echo ""
                        echo "ECR push successful."
                    '''
                }
            }
        }


        // ============================================================
        // 13. GET EXACT ECR IMAGE DIGEST
        // ============================================================

        stage('Get ECR Image Digest') {
            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    script {

                        env.ECR_IMAGE = sh(
                            script: '''
                                set -e

                                export AWS_DEFAULT_REGION="$AWS_REGION"

                                DIGEST=$(aws ecr describe-images \
                                    --repository-name "$ECR_REPOSITORY" \
                                    --image-ids imageTag="$BUILD_NUMBER" \
                                    --region "$AWS_REGION" \
                                    --query 'imageDetails[0].imageDigest' \
                                    --output text)

                                echo "$ECR_URI@$DIGEST"
                            ''',
                            returnStdout: true
                        ).trim()

                        echo ""
                        echo "=========================================="
                        echo "EXACT ECR IMAGE"
                        echo "=========================================="

                        echo "${env.ECR_IMAGE}"
                    }
                }
            }
        }


        // ============================================================
        // 14. VERIFY EC2 SSM CONNECTION
        // ============================================================

        stage('Verify EC2 SSM Connection') {
            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "VERIFYING EC2 SSM CONNECTION"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query 'InstanceInformationList[0].[InstanceId,PingStatus,AgentVersion]' \
                            --output table

                        PING_STATUS=$(aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query 'InstanceInformationList[0].PingStatus' \
                            --output text)

                        echo ""
                        echo "SSM Ping Status: $PING_STATUS"

                        if [ "$PING_STATUS" != "Online" ]; then
                            echo "EC2 is not Online in SSM."
                            exit 1
                        fi

                        echo ""
                        echo "EC2 SSM connection successful."
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
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "DEPLOYING APPLICATION TO EC2"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo ""
                        echo "EC2 Instance:"
                        echo "$EC2_INSTANCE_ID"

                        echo ""
                        echo "ECR Image:"
                        echo "$ECR_IMAGE"


                        # ==================================================
                        # Create deployment script
                        # ==================================================

                        cat > /tmp/ec2-deploy.sh <<EOF
#!/bin/bash

set -e

echo "=========================================="
echo "EC2 DEPLOYMENT STARTED"
echo "=========================================="

echo ""
echo "Checking Docker..."
docker --version

echo ""
echo "Checking AWS CLI..."
aws --version

echo ""
echo "ECR Registry:"
echo "$ECR_REGISTRY"

echo ""
echo "Application:"
echo "$APP_NAME"

echo ""
echo "Exact Image:"
echo "$ECR_IMAGE"


# ==================================================
# ECR LOGIN
# ==================================================

echo ""
echo "Logging into Amazon ECR..."

aws ecr get-login-password \
    --region "$AWS_REGION" |
    docker login \
    --username AWS \
    --password-stdin \
    "$ECR_REGISTRY"

echo "ECR login successful."


# ==================================================
# PULL EXACT IMAGE
# ==================================================

echo ""
echo "Pulling exact ECR image..."

docker pull "$ECR_IMAGE"


# ==================================================
# STOP OLD CONTAINER
# ==================================================

echo ""
echo "Stopping existing container..."

docker stop "$APP_NAME" || true


# ==================================================
# REMOVE OLD CONTAINER
# ==================================================

echo ""
echo "Removing existing container..."

docker rm "$APP_NAME" || true


# ==================================================
# START NEW CONTAINER
# ==================================================

echo ""
echo "Starting new container..."

docker run -d \
    --name "$APP_NAME" \
    --restart unless-stopped \
    -p "$HOST_PORT:$APP_PORT" \
    "$ECR_IMAGE"


# ==================================================
# WAIT FOR APPLICATION
# ==================================================

echo ""
echo "Waiting for application startup..."

sleep 5


# ==================================================
# VERIFY CONTAINER
# ==================================================

echo ""
echo "Container status:"

docker ps \
    --filter "name=$APP_NAME"


# ==================================================
# APPLICATION HEALTH CHECK
# ==================================================

echo ""
echo "Checking application..."

curl -f \
    "http://localhost:$APP_PORT/"


echo ""
echo "=========================================="
echo "EC2 DEPLOYMENT SUCCESSFUL"
echo "=========================================="
EOF


                        chmod +x /tmp/ec2-deploy.sh


                        # ==================================================
                        # Convert deployment script into SSM JSON
                        #
                        # IMPORTANT:
                        # We do NOT pass:
                        #
                        # commands=[echo ... | base64 -d ...]
                        #
                        # directly to AWS CLI.
                        #
                        # ==================================================

                        python3 - <<'PY' > /tmp/ssm-parameters.json

import json

with open("/tmp/ec2-deploy.sh", "r") as f:
    script = f.read()

parameters = {
    "commands": [
        "bash -c " + json.dumps(script)
    ]
}

print(json.dumps(parameters))

PY


                        echo ""
                        echo "SSM parameters created."


                        # ==================================================
                        # SEND COMMAND
                        # ==================================================

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "$EC2_INSTANCE_ID" \
                            --document-name "AWS-RunShellScript" \
                            --comment "DevSecOps deployment build $BUILD_NUMBER" \
                            --parameters file:///tmp/ssm-parameters.json \
                            --region "$AWS_REGION" \
                            --query "Command.CommandId" \
                            --output text)


                        echo ""
                        echo "SSM Command ID:"
                        echo "$COMMAND_ID"


                        # ==================================================
                        # WAIT
                        # ==================================================

                        echo ""
                        echo "Waiting for EC2 deployment..."

                        aws ssm wait command-executed \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION"


                        # ==================================================
                        # GET STATUS
                        # ==================================================

                        STATUS=$(aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "Status" \
                            --output text)


                        echo ""
                        echo "SSM Status:"
                        echo "$STATUS"


                        # ==================================================
                        # OUTPUT
                        # ==================================================

                        echo ""
                        echo "=========================================="
                        echo "EC2 STANDARD OUTPUT"
                        echo "=========================================="

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "StandardOutputContent" \
                            --output text


                        echo ""
                        echo "=========================================="
                        echo "EC2 STANDARD ERROR"
                        echo "=========================================="

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "StandardErrorContent" \
                            --output text


                        # ==================================================
                        # FAIL PIPELINE IF EC2 DEPLOYMENT FAILED
                        # ==================================================

                        if [ "$STATUS" != "Success" ]; then

                            echo ""
                            echo "EC2 deployment failed."

                            exit 1

                        fi


                        echo ""
                        echo "EC2 deployment completed successfully."
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
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "FINAL APPLICATION HEALTH CHECK"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="$AWS_REGION"


                        cat > /tmp/healthcheck.json <<EOF
{
    "commands": [
        "curl -f http://localhost:$APP_PORT/"
    ]
}
EOF


                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "$EC2_INSTANCE_ID" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Application health check build $BUILD_NUMBER" \
                            --parameters file:///tmp/healthcheck.json \
                            --region "$AWS_REGION" \
                            --query "Command.CommandId" \
                            --output text)


                        echo ""
                        echo "Health Check Command ID:"
                        echo "$COMMAND_ID"


                        aws ssm wait command-executed \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION"


                        STATUS=$(aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "Status" \
                            --output text)


                        echo ""
                        echo "Health Check Status:"
                        echo "$STATUS"


                        echo ""
                        echo "Health Check Output:"

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "StandardOutputContent" \
                            --output text


                        if [ "$STATUS" != "Success" ]; then

                            echo ""
                            echo "Application health check failed."

                            aws ssm get-command-invocation \
                                --command-id "$COMMAND_ID" \
                                --instance-id "$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query "StandardErrorContent" \
                                --output text

                            exit 1

                        fi


                        echo ""
                        echo "=========================================="
                        echo "APPLICATION IS HEALTHY"
                        echo "=========================================="
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
  ↓
Jenkins
  ↓
Gitleaks
  ↓
SonarQube
  ↓
NPM Audit
  ↓
Application Test
  ↓
Docker Build
  ↓
Trivy
  ↓
ECR
  ↓
SSM
  ↓
EC2
  ↓
Docker Container
  ↓
Health Check

APPLICATION DEPLOYED SUCCESSFULLY
==========================================
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
3. Node.js / npm
4. npm audit
5. Application test
6. Docker
7. Trivy
8. AWS credentials
9. ECR
10. SSM
11. EC2 Docker deployment
12. Application health check

==========================================
'''

        }


        always {

            sh '''
                rm -f \
                    /tmp/ec2-deploy.sh \
                    /tmp/ssm-parameters.json \
                    /tmp/healthcheck.json \
                    2>/dev/null || true
            '''

            echo "Pipeline execution completed."
        }
    }
}
