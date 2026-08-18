pipeline {

    agent any

    environment {

        // ============================================================
        // AWS
        // ============================================================
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '042775549160'

        // KEEP YOUR EXISTING JENKINS AWS CREDENTIAL ID HERE
        AWS_CREDENTIALS_ID = 'YOUR_EXISTING_AWS_CREDENTIAL_ID'

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
        // Application
        // ============================================================
        APP_NAME = 'devsecops-sample-app'
        APP_PORT = '3000'
        HOST_PORT = '3000'

        // ============================================================
        // SonarQube
        // ============================================================
        SONARQUBE_SERVER = 'sonarqube-dev-sec'

        // KEEP YOUR EXISTING SONAR TOKEN CREDENTIAL ID
        SONAR_TOKEN_CREDENTIAL_ID = 'YOUR_EXISTING_SONAR_TOKEN_CREDENTIAL_ID'
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

                        curl -fsS "$SONAR_HOST_URL/api/system/status"

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

                            sonar-scanner \
                                -Dsonar.token="$SONAR_TOKEN"

                            echo ""
                            echo "SonarQube scan completed."
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
                    echo "Application test stage completed."
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

                    docker build \
                        --pull \
                        -t "$ECR_URI:$BUILD_NUMBER" \
                        -t "$ECR_URI:latest" \
                        .

                    echo ""
                    echo "Docker images:"
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
                        echo "Pushing build image..."

                        docker push "$ECR_URI:$BUILD_NUMBER"

                        echo ""
                        echo "Pushing latest image..."

                        docker push "$ECR_URI:latest"

                        echo ""
                        echo "ECR push successful."
                    '''
                }
            }
        }

        // ============================================================
        // 13. GET EXACT ECR DIGEST
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

                        echo "Exact ECR image:"
                        echo "${env.ECR_IMAGE}"
                    }
                }
            }
        }

        // ============================================================
        // 14. VERIFY SSM
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

                        INSTANCE_INFO=$(aws ssm describe-instance-information \
                            --filters "Key=InstanceIds,Values=$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query 'InstanceInformationList[0].[InstanceId,PingStatus,AgentVersion]' \
                            --output text)

                        echo "$INSTANCE_INFO"

                        if [ -z "$INSTANCE_INFO" ] || \
                           echo "$INSTANCE_INFO" | grep -q "None"; then

                            echo "EC2 is not available through SSM."
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

                        echo "EC2:"
                        echo "$EC2_INSTANCE_ID"

                        echo ""
                        echo "ECR Image:"
                        echo "$ECR_IMAGE"

                        # ------------------------------------------------
                        # Create deployment script.
                        # This script will execute ON EC2.
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
echo "$ECR_REGISTRY"

echo ""
echo "Application:"
echo "$APP_NAME"

echo ""
echo "Image:"
echo "$ECR_IMAGE"

echo ""
echo "Logging into Amazon ECR..."

aws ecr get-login-password \
    --region "$AWS_REGION" |
docker login \
    --username AWS \
    --password-stdin \
    "$ECR_REGISTRY"

echo ""
echo "ECR login successful."

echo ""
echo "Pulling exact image by digest..."

docker pull "$ECR_IMAGE"

echo ""
echo "Stopping existing application..."

docker stop "$APP_NAME" || true

echo ""
echo "Removing existing application..."

docker rm "$APP_NAME" || true

echo ""
echo "Starting new application..."

docker run -d \
    --name "$APP_NAME" \
    --restart unless-stopped \
    --publish "$HOST_PORT:$APP_PORT" \
    "$ECR_IMAGE"

echo ""
echo "Waiting for application startup..."

sleep 5

echo ""
echo "Running containers:"

docker ps

echo ""
echo "Checking application endpoint..."

curl -f "http://localhost:$APP_PORT/"

echo ""
echo "=========================================="
echo "EC2 DEPLOYMENT SUCCESSFUL"
echo "=========================================="
EOF

                        chmod +x /tmp/ec2-deploy.sh

                        # ------------------------------------------------
                        # IMPORTANT:
                        #
                        # Do NOT construct:
                        #
                        # commands=[echo xxx | base64 -d ...]
                        #
                        # because shell operators get interpreted by the
                        # Jenkins shell and AWS CLI receives them as
                        # unknown options.
                        #
                        # Instead, create a JSON parameter file.
                        # ------------------------------------------------

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
                        echo "SSM parameter JSON created."

                        cat /tmp/ssm-parameters.json

                        echo ""
                        echo "Sending deployment command to EC2..."

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

                        echo ""
                        echo "Waiting for SSM deployment..."

                        aws ssm wait command-executed \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION"

                        echo ""
                        echo "Getting SSM execution status..."

                        STATUS=$(aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "Status" \
                            --output text)

                        echo "SSM Status:"
                        echo "$STATUS"

                        echo ""
                        echo "=========================================="
                        echo "REMOTE OUTPUT"
                        echo "=========================================="

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "StandardOutputContent" \
                            --output text

                        echo ""
                        echo "=========================================="
                        echo "REMOTE ERROR"
                        echo "=========================================="

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query "StandardErrorContent" \
                            --output text

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
        // 16. APPLICATION HEALTH CHECK
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
                        echo "APPLICATION HEALTH CHECK"
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
                            --comment "Application health check" \
                            --parameters file:///tmp/healthcheck.json \
                            --region "$AWS_REGION" \
                            --query "Command.CommandId" \
                            --output text)

                        echo "Health check command:"
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
3. npm dependencies
4. Application test
5. Docker
6. Trivy
7. AWS credentials
8. ECR
9. SSM
10. EC2 Docker deployment
11. Application health check

==========================================
'''
        }

        always {
            sh '''
                rm -f /tmp/ec2-deploy.sh \
                      /tmp/ssm-parameters.json \
                      /tmp/healthcheck.json \
                      2>/dev/null || true
            '''

            echo "Pipeline execution completed."
        }
    }
}
