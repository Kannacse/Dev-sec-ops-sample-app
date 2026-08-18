pipeline {

    agent any

    environment {

        // ============================================================
        // AWS
        // ============================================================

        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '042775549160'

        ECR_REPOSITORY = 'devsecops-sample-app'

        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"

        AWS_CREDENTIALS_ID = 'aws-hrms-v2-taff'


        // ============================================================
        // SONARQUBE
        // ============================================================

        SONARQUBE_SERVER = 'sonarqube-dev-sec'

        SONAR_TOKEN_CREDENTIAL_ID = 'sonarqube-jenkins-token'

        SONAR_SCANNER_HOME = '/opt/sonar-scanner'


        // ============================================================
        // EC2 / SSM
        // ============================================================

        EC2_INSTANCE_ID = 'YOUR_EC2_INSTANCE_ID'

        EC2_PORT = '3000'


        // ============================================================
        // APPLICATION
        // ============================================================

        APP_NAME = 'devsecops-sample-app'

        DOCKER_CONTAINER_NAME = 'devsecops-sample-app'

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
                    echo "Gitleaks scan passed."
                    echo "No secrets detected."
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

                        echo ""
                        echo "SonarQube URL:"
                        echo "$SONAR_HOST_URL"

                        echo ""
                        echo "Checking SonarQube..."

                        curl -fsS "$SONAR_HOST_URL/api/system/status"

                        echo ""
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

                            "${SONAR_SCANNER_HOME}/bin/sonar-scanner" --version

                            echo ""
                            echo "Running SonarQube SAST..."

                            "${SONAR_SCANNER_HOME}/bin/sonar-scanner" \
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
                    echo "INSTALL NODE.JS DEPENDENCIES"
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
        // WARNING ONLY
        // ============================================================

        stage('Dependency Audit') {

            steps {

                sh '''
                    echo "=========================================="
                    echo "NPM DEPENDENCY AUDIT"
                    echo "=========================================="

                    echo ""
                    echo "Running npm audit..."

                    set +e

                    npm audit --audit-level=high

                    AUDIT_EXIT_CODE=$?

                    set -e

                    echo ""
                    echo "npm audit exit code: $AUDIT_EXIT_CODE"

                    if [ "$AUDIT_EXIT_CODE" -ne 0 ]; then
                        echo ""
                        echo "WARNING: npm audit reported vulnerabilities."
                        echo "Pipeline will continue."
                    else
                        echo ""
                        echo "npm audit passed."
                    fi

                    echo ""
                    echo "Dependency audit stage completed."
                '''
            }
        }


        // ============================================================
        // 8. APPLICATION TEST
        // WARNING ONLY WHEN NO REAL TEST IS CONFIGURED
        // ============================================================

        stage('Application Test') {

            steps {

                sh '''
                    set -e

                    echo "=========================================="
                    echo "APPLICATION TEST"
                    echo "=========================================="

                    TEST_SCRIPT=$(node -p "require('./package.json').scripts?.test || ''")

                    echo ""
                    echo "Configured test script:"
                    echo "$TEST_SCRIPT"

                    if [ -z "$TEST_SCRIPT" ]; then

                        echo ""
                        echo "No test script is configured."
                        echo "Skipping application tests."

                    elif echo "$TEST_SCRIPT" | grep -q "no test specified"; then

                        echo ""
                        echo "Placeholder test script detected."
                        echo "No automated application tests are configured."
                        echo "Skipping application tests."

                    else

                        echo ""
                        echo "Running application tests..."

                        npm test

                        echo ""
                        echo "Application tests passed."

                    fi

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
                    echo "DOCKER IMAGE BUILD"
                    echo "=========================================="

                    echo ""
                    echo "Docker version:"
                    docker --version

                    echo ""
                    echo "Building image..."

                    docker build \
                        -t "$ECR_URI:$BUILD_NUMBER" \
                        -t "$ECR_URI:latest" \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    docker images | grep "$ECR_REPOSITORY" || true
                '''
            }
        }


        // ============================================================
        // 10. TRIVY CONTAINER SCAN
        // WARNING ONLY
        // ============================================================

        stage('Container Scan - Trivy') {

            steps {

                sh '''
                    echo "=========================================="
                    echo "TRIVY CONTAINER SECURITY SCAN"
                    echo "=========================================="

                    echo ""
                    echo "Checking Trivy..."

                    trivy --version

                    echo ""
                    echo "Image:"
                    echo "$ECR_URI:$BUILD_NUMBER"

                    echo ""
                    echo "=========================================="
                    echo "TRIVY POLICY"
                    echo "=========================================="

                    echo "Severity: HIGH, CRITICAL"
                    echo "Mode: WARNING ONLY"
                    echo "Pipeline will continue"
                    echo "=========================================="

                    echo ""

                    set +e

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        "$ECR_URI:$BUILD_NUMBER"

                    TRIVY_EXIT_CODE=$?

                    set -e

                    echo ""
                    echo "=========================================="
                    echo "TRIVY RESULT"
                    echo "=========================================="

                    echo "Trivy exit code: $TRIVY_EXIT_CODE"

                    if [ "$TRIVY_EXIT_CODE" -eq 0 ]; then

                        echo ""
                        echo "Trivy scan completed successfully."

                    else

                        echo ""
                        echo "WARNING: Trivy returned exit code $TRIVY_EXIT_CODE."
                        echo "Pipeline will continue."

                    fi

                    echo ""
                    echo "Trivy stage completed."

                    exit 0
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
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "AWS CONNECTION TEST"
                        echo "=========================================="

                        aws --version

                        echo ""
                        echo "AWS Region:"
                        echo "$AWS_REGION"

                        echo ""
                        echo "Testing AWS identity..."

                        aws sts get-caller-identity

                        echo ""
                        echo "AWS connection successful."
                    '''
                }
            }
        }


        // ============================================================
        // 12. LOGIN TO ECR
        // ============================================================

        stage('ECR Login') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "ECR LOGIN"
                        echo "=========================================="

                        aws ecr get-login-password \
                            --region "$AWS_REGION" \
                        | docker login \
                            --username AWS \
                            --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

                        echo ""
                        echo "ECR login successful."
                    '''
                }
            }
        }


        // ============================================================
        // 13. PUSH IMAGE TO ECR
        // ============================================================

        stage('Push Image to ECR') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "PUSH IMAGE TO ECR"
                        echo "=========================================="

                        docker push "$ECR_URI:$BUILD_NUMBER"

                        echo ""
                        echo "Pushing latest tag..."

                        docker push "$ECR_URI:latest"

                        echo ""
                        echo "Image pushed successfully."
                    '''
                }
            }
        }


        // ============================================================
        // 14. GET ECR IMAGE DIGEST
        // ============================================================

        stage('Get ECR Image Digest') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "GET ECR IMAGE DIGEST"
                        echo "=========================================="

                        IMAGE_DIGEST=$(aws ecr describe-images \
                            --repository-name "$ECR_REPOSITORY" \
                            --image-ids imageTag="$BUILD_NUMBER" \
                            --region "$AWS_REGION" \
                            --query 'imageDetails[0].imageDigest' \
                            --output text)

                        echo ""
                        echo "ECR Image Digest:"
                        echo "$IMAGE_DIGEST"

                        test -n "$IMAGE_DIGEST"

                        echo ""
                        echo "ECR image digest retrieved successfully."
                    '''
                }
            }
        }


        // ============================================================
        // 15. VERIFY EC2 SSM CONNECTION
        // ============================================================

        stage('Verify EC2 SSM Connection') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "VERIFY EC2 SSM CONNECTION"
                        echo "=========================================="

                        echo ""
                        echo "Checking EC2 instance:"
                        echo "$EC2_INSTANCE_ID"

                        aws ssm describe-instance-information \
                            --region "$AWS_REGION"

                        echo ""
                        echo "Checking target instance..."

                        INSTANCE_STATE=$(aws ec2 describe-instances \
                            --instance-ids "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query 'Reservations[0].Instances[0].State.Name' \
                            --output text)

                        echo "EC2 state: $INSTANCE_STATE"

                        if [ "$INSTANCE_STATE" != "running" ]; then
                            echo "ERROR: EC2 instance is not running."
                            exit 1
                        fi

                        echo ""
                        echo "EC2 instance is running."
                    '''
                }
            }
        }


        // ============================================================
        // 16. DEPLOY TO EC2 VIA SSM
        // ============================================================

        stage('Deploy to EC2 via SSM') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "DEPLOY TO EC2 VIA SSM"
                        echo "=========================================="

                        cat > /tmp/ec2-deploy.sh <<'EOF'
#!/bin/bash

set -e

echo "Starting EC2 deployment..."

echo "Logging into ECR..."

aws ecr get-login-password \
    --region us-east-1 \
| docker login \
    --username AWS \
    --password-stdin 042775549160.dkr.ecr.us-east-1.amazonaws.com

echo "Pulling application image..."

docker pull 042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app:${BUILD_NUMBER}

echo "Stopping existing container..."

docker rm -f devsecops-sample-app 2>/dev/null || true

echo "Starting new container..."

docker run -d \
    --name devsecops-sample-app \
    --restart unless-stopped \
    -p 3000:3000 \
    042775549160.dkr.ecr.us-east-1.amazonaws.com/devsecops-sample-app:${BUILD_NUMBER}

echo "Checking container..."

docker ps | grep devsecops-sample-app

echo "Deployment completed."

EOF

                        sed -i "s/\${BUILD_NUMBER}/$BUILD_NUMBER/g" /tmp/ec2-deploy.sh

                        cat > /tmp/ssm-parameters.json <<EOF
{
    "commands": [
        "$(sed 's/"/\\"/g' /tmp/ec2-deploy.sh | sed ':a;N;$!ba;s/\\n/","/g')"
    ]
}
EOF

                        echo ""
                        echo "Sending deployment command to EC2..."

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "$EC2_INSTANCE_ID" \
                            --document-name "AWS-RunShellScript" \
                            --parameters file:///tmp/ssm-parameters.json \
                            --region "$AWS_REGION" \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
                        echo "SSM Command ID:"
                        echo "$COMMAND_ID"

                        echo ""
                        echo "Waiting for deployment..."

                        aws ssm wait command-executed \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION"

                        echo ""
                        echo "Reading deployment result..."

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query '{Status:Status,Output:StandardOutputContent,Error:StandardErrorContent}' \
                            --output json

                        COMMAND_STATUS=$(aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query 'Status' \
                            --output text)

                        echo ""
                        echo "SSM command status:"
                        echo "$COMMAND_STATUS"

                        if [ "$COMMAND_STATUS" != "Success" ]; then
                            echo ""
                            echo "ERROR: EC2 deployment failed."
                            exit 1
                        fi

                        echo ""
                        echo "EC2 deployment successful."
                    '''
                }
            }
        }


        // ============================================================
        // 17. APPLICATION HEALTH CHECK
        // ============================================================

        stage('Application Health Check') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS_ID}"]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "APPLICATION HEALTH CHECK"
                        echo "=========================================="

                        echo ""
                        echo "Checking application through SSM..."

                        cat > /tmp/healthcheck.json <<'EOF'
{
    "commands": [
        "curl -f http://localhost:3000 || exit 1"
    ]
}
EOF

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "$EC2_INSTANCE_ID" \
                            --document-name "AWS-RunShellScript" \
                            --parameters file:///tmp/healthcheck.json \
                            --region "$AWS_REGION" \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
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
                            --query 'Status' \
                            --output text)

                        echo ""
                        echo "Health check status:"
                        echo "$STATUS"

                        aws ssm get-command-invocation \
                            --command-id "$COMMAND_ID" \
                            --instance-id "$EC2_INSTANCE_ID" \
                            --region "$AWS_REGION" \
                            --query '{Output:StandardOutputContent,Error:StandardErrorContent}' \
                            --output json

                        if [ "$STATUS" != "Success" ]; then
                            echo ""
                            echo "ERROR: Application health check failed."
                            exit 1
                        fi

                        echo ""
                        echo "Application health check passed."
                    '''
                }
            }
        }
    }


    // ================================================================
    // POST ACTIONS
    // ================================================================

    post {

        always {

            sh '''
                rm -f \
                    /tmp/ec2-deploy.sh \
                    /tmp/ssm-parameters.json \
                    /tmp/healthcheck.json
            '''

            echo "Pipeline execution completed."
        }


        success {

            echo '''

==========================================
DEVSECOPS PIPELINE SUCCESS
==========================================

All required stages completed successfully.

Security:
- Gitleaks
- SonarQube
- npm audit
- Trivy

Build:
- Docker

AWS:
- ECR
- EC2
- SSM

Deployment:
- Application deployed
- Health check passed

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
    }
}
