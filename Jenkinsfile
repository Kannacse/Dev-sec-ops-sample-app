// ================================================================
// STAGE NOTIFICATION HELPER
// Sends one notification per stage to SES/Gmail and Google Chat.
// ================================================================

def notifyStage(String stageName, String stageStatus, String stageOutput) {

    def safeOutput = (stageOutput ?: "No stage output captured.")
        .take(12000)

    def payload = [
        notification_type: "STAGE",
        status           : stageStatus,
        job              : env.JOB_NAME,
        build            : env.BUILD_NUMBER,
        build_url        : env.BUILD_URL,
        stage            : stageName,
        stage_output     : safeOutput
    ]

    def payloadJson = groovy.json.JsonOutput.toJson(payload)

    writeFile(
        file: "/tmp/stage-notification-${BUILD_NUMBER}.json",
        text: payloadJson
    )

    try {
        withEnv(["STAGE_NAME_FOR_NOTIFY=${stageName}", "STAGE_STATUS_FOR_NOTIFY=${stageStatus}"]) {
            withCredentials([
                [
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIALS_ID}"
                ]
            ]) {
                sh '''
            set -e

            echo ""
            echo "=========================================="
            echo "STAGE NOTIFICATION"
            echo "=========================================="
            echo "Stage: ${STAGE_NAME_FOR_NOTIFY}"
            echo "Status: ${STAGE_STATUS_FOR_NOTIFY}"
            echo ""
            echo "Invoking notification Lambda..."

            aws lambda invoke \
                --function-name devsecops-pipeline-notification \
                --region "${AWS_REGION}" \
                --cli-binary-format raw-in-base64-out \
                --payload fileb:///tmp/stage-notification-${BUILD_NUMBER}.json \
                /tmp/stage-notification-response-${BUILD_NUMBER}.json

            echo ""
            echo "Lambda response:"
            cat /tmp/stage-notification-response-${BUILD_NUMBER}.json
                '''
            }
        }
    } catch (Exception e) {
        echo "WARNING: Stage notification failed for ${stageName}: ${e.getMessage()}"
    }
}

pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
    }

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

        EC2_INSTANCE_ID = 'i-096fc3c14a9db3ad8'

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
                script {
                    def stageName = "Checkout"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-1.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    checkout scm
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 2. VERIFY SOURCE
        // ============================================================

        stage('Verify Source') {

            steps {
                script {
                    def stageName = "Verify Source"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-2.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

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
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 3. GITLEAKS SECRET SCAN
        // ============================================================

        stage('Secret Scan - Gitleaks') {

            steps {
                script {
                    def stageName = "Secret Scan - Gitleaks"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-3.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

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
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 4. SONARQUBE CONNECTION TEST
        // ============================================================

        stage('SonarQube Connection Test') {

            steps {
                script {
                    def stageName = "SonarQube Connection Test"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-4.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withSonarQubeEnv("${SONARQUBE_SERVER}") {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "SONARQUBE CONNECTION TEST"
                            echo "=========================================="

                            echo ""
                            echo "SonarQube URL:"
                            echo "$SONAR_HOST_URL"

                            echo ""
                            echo "Checking SonarQube..."

                            curl -fsS \
                                "$SONAR_HOST_URL/api/system/status"

                            echo ""
                            echo ""
                            echo "SonarQube connection successful."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 5. SONARQUBE SAST
        // ============================================================

        stage('SAST - SonarQube') {

            steps {
                script {
                    def stageName = "SAST - SonarQube"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-5.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withSonarQubeEnv("${SONARQUBE_SERVER}") {

                        withCredentials([
                            string(
                                credentialsId: "${SONAR_TOKEN_CREDENTIAL_ID}",
                                variable: 'SONAR_TOKEN'
                            )
                        ]) {

                            sh '''
                            set -o pipefail
                            {

                                set -e

                                echo "=========================================="
                                echo "SONARQUBE SAST SCAN"
                                echo "=========================================="

                                echo ""
                                echo "Checking SonarScanner..."

                                test -x "${SONAR_SCANNER_HOME}/bin/sonar-scanner"

                                "${SONAR_SCANNER_HOME}/bin/sonar-scanner" \
                                    --version

                                echo ""
                                echo "Running SonarQube SAST..."

                                "${SONAR_SCANNER_HOME}/bin/sonar-scanner" \
                                    -Dsonar.token="$SONAR_TOKEN"

                                echo ""
                                echo "SonarQube SAST scan completed."
                        
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                        }
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 6. INSTALL DEPENDENCIES
        // ============================================================

        stage('Install Dependencies') {

            steps {
                script {
                    def stageName = "Install Dependencies"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-6.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

                        set -e

                        echo "=========================================="
                        echo "INSTALL NODE.JS DEPENDENCIES"
                        echo "=========================================="

                        echo ""
                        echo "Node version:"
                        node --version

                        echo ""
                        echo "NPM version:"
                        npm --version

                        echo ""
                        echo "Installing dependencies..."

                        npm ci

                        echo ""
                        echo "Dependencies installed successfully."
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 7. DEPENDENCY AUDIT
        // WARNING ONLY
        // ============================================================

        stage('Dependency Audit') {

            steps {
                script {
                    def stageName = "Dependency Audit"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-7.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

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
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 8. APPLICATION TEST
        // ============================================================

        stage('Application Test') {

            steps {
                script {
                    def stageName = "Application Test"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-8.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

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

                        elif echo "$TEST_SCRIPT" | grep -qi "no test specified"; then

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
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 9. DOCKER BUILD
        // ============================================================

        stage('Docker Build') {

            steps {
                script {
                    def stageName = "Docker Build"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-9.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

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

                        echo ""
                        echo "Built images:"

                        docker images | grep "$ECR_REPOSITORY" || true
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 10. TRIVY CONTAINER SCAN
        // WARNING ONLY
        // ============================================================

        stage('Container Scan - Trivy') {

            steps {
                script {
                    def stageName = "Container Scan - Trivy"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-10.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    sh '''
                            set -o pipefail
                            {

                        set -e

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
                        echo "Pipeline will continue if vulnerabilities are found."

                        echo ""
                        echo "=========================================="
                        echo "RUNNING TRIVY"
                        echo "=========================================="

                        trivy image \
                            --scanners vuln \
                            --severity HIGH,CRITICAL \
                            --exit-code 0 \
                            "$ECR_URI:$BUILD_NUMBER"

                        echo ""
                        echo "=========================================="
                        echo "TRIVY RESULT"
                        echo "=========================================="

                        echo "Trivy scan completed."
                        echo "HIGH/CRITICAL findings are reported as warnings."
                        echo "Pipeline will continue."

                        echo ""
                        echo "Trivy stage completed successfully."
                
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 11. AWS CONNECTION TEST
        // ============================================================

        stage('AWS Connection Test') {

            steps {
                script {
                    def stageName = "AWS Connection Test"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-11.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "AWS CONNECTION TEST"
                            echo "=========================================="

                            echo ""
                            echo "AWS CLI version:"
                            aws --version

                            echo ""
                            echo "AWS Region:"
                            echo "$AWS_REGION"

                            echo ""
                            echo "Testing AWS identity..."

                            aws sts get-caller-identity

                            echo ""
                            echo "AWS connection successful."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 12. ECR LOGIN
        // ============================================================

        stage('ECR Login') {

            steps {
                script {
                    def stageName = "ECR Login"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-12.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "ECR LOGIN"
                            echo "=========================================="

                            aws ecr get-login-password \
                                --region "$AWS_REGION" \
                            | docker login \
                                --username AWS \
                                --password-stdin \
                                "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

                            echo ""
                            echo "ECR login successful."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 13. PUSH IMAGE TO ECR
        // ============================================================

        stage('Push Image to ECR') {

            steps {
                script {
                    def stageName = "Push Image to ECR"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-13.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "PUSH IMAGE TO ECR"
                            echo "=========================================="

                            echo ""
                            echo "Pushing build image..."

                            docker push \
                                "$ECR_URI:$BUILD_NUMBER"

                            echo ""
                            echo "Pushing latest image..."

                            docker push \
                                "$ECR_URI:latest"

                            echo ""
                            echo "Image pushed successfully."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 14. GET ECR IMAGE DIGEST
        // ============================================================

        stage('Get ECR Image Digest') {

            steps {
                script {
                    def stageName = "Get ECR Image Digest"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-14.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

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
                            test "$IMAGE_DIGEST" != "None"

                            echo ""
                            echo "ECR image digest retrieved successfully."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 15. VERIFY EC2 SSM CONNECTION
        // ============================================================

        stage('Verify EC2 SSM Connection') {

            steps {
                script {
                    def stageName = "Verify EC2 SSM Connection"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-15.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "VERIFY EC2 SSM CONNECTION"
                            echo "=========================================="

                            echo ""
                            echo "EC2 Instance:"
                            echo "$EC2_INSTANCE_ID"

                            echo ""
                            echo "Checking EC2 state..."

                            INSTANCE_STATE=$(aws ec2 describe-instances \
                                --instance-ids "$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query 'Reservations[0].Instances[0].State.Name' \
                                --output text)

                            echo "EC2 state: $INSTANCE_STATE"

                            if [ "$INSTANCE_STATE" != "running" ]; then

                                echo ""
                                echo "ERROR: EC2 instance is not running."
                                exit 1

                            fi

                            echo ""
                            echo "Checking SSM agent..."

                            SSM_STATUS=$(aws ssm describe-instance-information \
                                --filters "Key=InstanceIds,Values=$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query 'InstanceInformationList[0].PingStatus' \
                                --output text)

                            echo "SSM status: $SSM_STATUS"

                            if [ "$SSM_STATUS" != "Online" ]; then

                                echo ""
                                echo "ERROR: EC2 instance is not online in SSM."
                                exit 1

                            fi

                            echo ""
                            echo "EC2 and SSM connection verified successfully."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 16. DEPLOY TO EC2 VIA SSM
        // ============================================================

        stage('Deploy to EC2 via SSM') {

            steps {
                script {
                    def stageName = "Deploy to EC2 via SSM"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-16.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "DEPLOY TO EC2 VIA SSM"
                            echo "=========================================="

                            echo ""
                            echo "Creating deployment script..."

                            cat > /tmp/ec2-deploy.sh <<EOF
    #!/bin/bash

    set -e

    echo "=========================================="
    echo "EC2 DEPLOYMENT"
    echo "=========================================="

    echo ""
    echo "Logging into ECR..."

    aws ecr get-login-password \
        --region ${AWS_REGION} \
    | docker login \
        --username AWS \
        --password-stdin \
        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

    echo ""
    echo "Pulling application image..."

    docker pull \
        ${ECR_URI}:${BUILD_NUMBER}

    echo ""
    echo "Stopping old container..."

    docker rm -f ${DOCKER_CONTAINER_NAME} 2>/dev/null || true

    echo ""
    echo "Starting new container..."

    docker run -d \
        --name ${DOCKER_CONTAINER_NAME} \
        --restart unless-stopped \
        -p ${EC2_PORT}:${EC2_PORT} \
        -e PORT=${EC2_PORT} \
        ${ECR_URI}:${BUILD_NUMBER}

    echo ""
    echo "Waiting for application..."

    sleep 5

    echo ""
    echo "Checking container..."

    docker ps \
        --filter "name=${DOCKER_CONTAINER_NAME}"

    echo ""
    echo "Checking application locally..."

    curl -fsS \
        http://127.0.0.1:${EC2_PORT}/

    echo ""
    echo ""
    echo "EC2 deployment successful."

    EOF

                            chmod +x /tmp/ec2-deploy.sh

                            echo ""
                            echo "Encoding deployment script..."

                            DEPLOY_SCRIPT_B64=$(base64 -w0 /tmp/ec2-deploy.sh)

                            echo ""
                            echo "Creating SSM parameter JSON..."

                            python3 - "$DEPLOY_SCRIPT_B64" > /tmp/ssm-parameters.json <<'PY'
    import json
    import sys

    script_b64 = sys.argv[1]

    commands = [
        f"echo {script_b64} | base64 -d > /tmp/ec2-deploy.sh",
        "chmod +x /tmp/ec2-deploy.sh",
        "sudo /tmp/ec2-deploy.sh"
    ]

    print(json.dumps({
        "commands": commands
    }))
    PY

                            echo ""
                            echo "Validating SSM parameter JSON..."

                            python3 -m json.tool \
                                /tmp/ssm-parameters.json

                            echo ""
                            echo "Sending deployment command to EC2..."

                            SSM_COMMAND_ID=$(aws ssm send-command \
                                --instance-ids "$EC2_INSTANCE_ID" \
                                --document-name "AWS-RunShellScript" \
                                --comment "DevSecOps deployment - build $BUILD_NUMBER" \
                                --parameters file:///tmp/ssm-parameters.json \
                                --region "$AWS_REGION" \
                                --query 'Command.CommandId' \
                                --output text)

                            echo ""
                            echo "SSM Command ID:"
                            echo "$SSM_COMMAND_ID"

                            test -n "$SSM_COMMAND_ID"

                            echo ""
                            echo "Waiting for deployment..."

                            COMMAND_STATUS="Pending"

                            for i in $(seq 1 36); do

                                COMMAND_STATUS=$(aws ssm get-command-invocation \
                                    --command-id "$SSM_COMMAND_ID" \
                                    --instance-id "$EC2_INSTANCE_ID" \
                                    --region "$AWS_REGION" \
                                    --query 'Status' \
                                    --output text 2>/dev/null || true)

                                echo "Attempt $i/36 - Status: $COMMAND_STATUS"

                                case "$COMMAND_STATUS" in

                                    Success)
                                        break
                                        ;;

                                    Failed|Cancelled|TimedOut|Cancelling)
                                        break
                                        ;;

                                esac

                                sleep 5

                            done

                            echo ""
                            echo "=========================================="
                            echo "SSM DEPLOYMENT RESULT"
                            echo "=========================================="

                            echo ""
                            echo "Status:"
                            echo "$COMMAND_STATUS"

                            echo ""
                            echo "Command output:"

                            aws ssm get-command-invocation \
                                --command-id "$SSM_COMMAND_ID" \
                                --instance-id "$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query 'StandardOutputContent' \
                                --output text

                            echo ""
                            echo "Command error output:"

                            aws ssm get-command-invocation \
                                --command-id "$SSM_COMMAND_ID" \
                                --instance-id "$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query 'StandardErrorContent' \
                                --output text

                            if [ "$COMMAND_STATUS" != "Success" ]; then

                                echo ""
                                echo "ERROR: EC2 deployment failed."
                                exit 1

                            fi

                            echo ""
                            echo "=========================================="
                            echo "EC2 DEPLOYMENT SUCCESSFUL"
                            echo "=========================================="
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }


        // ============================================================
        // 17. APPLICATION HEALTH CHECK
        // ============================================================

        stage('Application Health Check') {

            steps {
                script {
                    def stageName = "Application Health Check"
                    def stageOutputFile = "/tmp/jenkins-stage-${BUILD_NUMBER}-17.log"
                    try {
                        withEnv(["STAGE_OUTPUT_FILE=${stageOutputFile}"] ) {


                    withCredentials([
                        [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: "${AWS_CREDENTIALS_ID}"
                        ]
                    ]) {

                        sh '''
                            set -o pipefail
                            {

                            set -e

                            echo "=========================================="
                            echo "APPLICATION HEALTH CHECK"
                            echo "=========================================="

                            echo ""
                            echo "Creating health-check parameters..."

                            python3 > /tmp/healthcheck-parameters.json <<PY
    import json

    print(json.dumps({
        "commands": [
            "curl -fsS http://127.0.0.1:${EC2_PORT}/"
        ]
    }))
    PY

                            echo ""
                            echo "Health-check parameters:"

                            python3 -m json.tool \
                                /tmp/healthcheck-parameters.json

                            echo ""
                            echo "Running health check through SSM..."

                            HEALTH_COMMAND_ID=$(aws ssm send-command \
                                --instance-ids "$EC2_INSTANCE_ID" \
                                --document-name "AWS-RunShellScript" \
                                --comment "Application health check - build $BUILD_NUMBER" \
                                --parameters file:///tmp/healthcheck-parameters.json \
                                --region "$AWS_REGION" \
                                --query 'Command.CommandId' \
                                --output text)

                            echo ""
                            echo "Health Check Command ID:"
                            echo "$HEALTH_COMMAND_ID"

                            test -n "$HEALTH_COMMAND_ID"

                            echo ""
                            echo "Waiting for health check..."

                            HEALTH_STATUS="Pending"

                            for i in $(seq 1 24); do

                                HEALTH_STATUS=$(aws ssm get-command-invocation \
                                    --command-id "$HEALTH_COMMAND_ID" \
                                    --instance-id "$EC2_INSTANCE_ID" \
                                    --region "$AWS_REGION" \
                                    --query 'Status' \
                                    --output text 2>/dev/null || true)

                                echo "Attempt $i/24 - Status: $HEALTH_STATUS"

                                case "$HEALTH_STATUS" in

                                    Success)
                                        break
                                        ;;

                                    Failed|Cancelled|TimedOut|Cancelling)
                                        break
                                        ;;

                                esac

                                sleep 5

                            done

                            echo ""
                            echo "=========================================="
                            echo "HEALTH CHECK RESULT"
                            echo "=========================================="

                            echo ""
                            echo "Status:"
                            echo "$HEALTH_STATUS"

                            echo ""
                            echo "Application output:"

                            aws ssm get-command-invocation \
                                --command-id "$HEALTH_COMMAND_ID" \
                                --instance-id "$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query 'StandardOutputContent' \
                                --output text

                            echo ""
                            echo "Application error output:"

                            aws ssm get-command-invocation \
                                --command-id "$HEALTH_COMMAND_ID" \
                                --instance-id "$EC2_INSTANCE_ID" \
                                --region "$AWS_REGION" \
                                --query 'StandardErrorContent' \
                                --output text

                            if [ "$HEALTH_STATUS" != "Success" ]; then

                                echo ""
                                echo "ERROR: Application health check failed."

                                exit 1

                            fi

                            echo ""
                            echo "=========================================="
                            echo "APPLICATION HEALTH CHECK PASSED"
                            echo "=========================================="

                            echo ""
                            echo "Application is healthy on EC2."
                    
                            } 2>&1 | tee "$STAGE_OUTPUT_FILE"
                    
                    }
            
                        }

                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : "Stage completed successfully."
                        notifyStage(stageName, "SUCCESS", stageOutput)

                    } catch (Exception e) {
                        def stageOutput = fileExists(stageOutputFile) ? readFile(stageOutputFile) : (e.getMessage() ?: "Stage failed before output was captured.")
                        notifyStage(stageName, "FAILURE", stageOutput)
                        throw e
                    }
                }
            }
        }
    }


// ================================================================
// POST ACTIONS
// ================================================================

post {

    always {

        echo ""
        echo "=========================================="
        echo "PIPELINE EXECUTION COMPLETED"
        echo "=========================================="

        script {
            echo "Stage notifications were sent after each pipeline stage."
        }


        // ============================================================
        // CLEAN TEMPORARY FILES
        // ============================================================

        sh '''
            rm -f \
                /tmp/ec2-deploy.sh \
                /tmp/ssm-parameters.json \
                /tmp/healthcheck-parameters.json \
                /tmp/healthcheck.json \
                /tmp/lambda-payload.json \
                /tmp/lambda-response.json \
                /tmp/stage-notification-${BUILD_NUMBER}.json \
                /tmp/stage-notification-response-${BUILD_NUMBER}.json \
                /tmp/jenkins-stage-${BUILD_NUMBER}-*.log \
                2>/dev/null || true
        '''


        echo ""
        echo "=========================================="
        echo "POST ACTIONS COMPLETED"
        echo "=========================================="
    }


    success {

        echo ""
        echo "=========================================="
        echo "DEVSECOPS PIPELINE SUCCESS"
        echo "=========================================="

        echo ""
        echo "Build Number:"
        echo "${BUILD_NUMBER}"

        echo ""
        echo "Docker Image:"
        echo "${ECR_URI}:${BUILD_NUMBER}"

        echo ""
        echo "Deployment:"
        echo "EC2 via AWS SSM"

        echo ""
        echo "Security:"
        echo "Gitleaks - Passed"
        echo "SonarQube - Completed"
        echo "npm Audit - Completed"
        echo "Trivy - Warning Only"

        echo ""
        echo "Application:"
        echo "Health Check Passed"

        echo ""
        echo "=========================================="
    }


    failure {

        echo ""
        echo "=========================================="
        echo "DEVSECOPS PIPELINE FAILED"
        echo "=========================================="

        echo ""
        echo "Check the FIRST failed stage in Console Output."

        echo ""
        echo "Possible areas:"

        echo "1. Gitleaks"
        echo "2. SonarQube"
        echo "3. npm dependencies"
        echo "4. Application test"
        echo "5. Docker"
        echo "6. Trivy"
        echo "7. AWS credentials"
        echo "8. ECR"
        echo "9. SSM"
        echo "10. EC2 deployment"
        echo "11. Application health check"

        echo ""
        echo "=========================================="
    }
}

}
