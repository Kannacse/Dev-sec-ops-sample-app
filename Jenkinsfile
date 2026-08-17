pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        EC2_INSTANCE_ID = 'i-0e0aebd33e4c8e9a1'

        DOCKER_IMAGE = 'kannancloud/devsecops-sample-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_IMAGE = "kannancloud/devsecops-sample-app:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Source') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Verifying Application Source"
                    echo "=========================================="

                    ls -la

                    test -f app.js
                    test -f package.json
                    test -f Dockerfile
                    test -f sonar-project.properties

                    echo "Source verification successful."
                '''
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Gitleaks Secret Scan"
                    echo "=========================================="

                    gitleaks detect \
                        --source . \
                        --no-banner

                    echo "Gitleaks scan completed successfully."
                '''
            }
        }

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

                            echo "SonarQube analysis completed successfully."
                        '''
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Installing Application Dependencies"
                    echo "=========================================="

                    npm install

                    echo "Dependencies installed successfully."
                '''
            }
        }

        stage('Dependency Audit') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running NPM Dependency Audit"
                    echo "=========================================="

                    npm audit --audit-level=high || true

                    echo ""
                    echo "WARNING: NPM audit completed."
                    echo "Review dependency vulnerabilities before production deployment."
                    echo "Pipeline will continue for this development environment."
                '''
            }
        }

        stage('Application Test') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Application Test"
                    echo "=========================================="

                    node --check app.js

                    echo "Application syntax check successful."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Building Docker Image"
                    echo "=========================================="

                    docker build \
                        -t ${FULL_IMAGE} \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    docker images | grep devsecops-sample-app
                '''
            }
        }

        stage('Container Scan - Trivy') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Running Trivy Container Security Scan"
                    echo "=========================================="

                    echo "Scanning image:"
                    echo "${FULL_IMAGE}"

                    trivy image \
                        --severity HIGH,CRITICAL \
                        "${FULL_IMAGE}" || true

                    echo ""
                    echo "WARNING: Trivy scan completed."
                    echo "HIGH/CRITICAL vulnerabilities may require remediation."
                    echo "Pipeline will continue for this development environment."
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Docker Hub Credentials',
                        usernameVariable: 'DOCKERHUB_USERNAME',
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "=========================================="
                        echo "Pushing Docker Image to Docker Hub"
                        echo "=========================================="

                        echo "$DOCKERHUB_PASSWORD" | docker login \
                            --username "$DOCKERHUB_USERNAME" \
                            --password-stdin

                        docker push "${FULL_IMAGE}"

                        docker logout

                        echo ""
                        echo "Docker image pushed successfully:"
                        echo "${FULL_IMAGE}"
                    '''
                }
            }
        }

        stage('AWS Connection Test') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-hrms-v2-taff']
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

        stage('Deploy to EC2 via SSM') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-hrms-v2-taff']
                ]) {
                    sh '''
                        echo "=========================================="
                        echo "Deploying Application to EC2"
                        echo "=========================================="

                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        echo "EC2 Instance:"
                        echo "${EC2_INSTANCE_ID}"

                        echo "Docker Image:"
                        echo "${FULL_IMAGE}"

                        COMMAND_ID=$(aws ssm send-command \
                            --instance-ids "${EC2_INSTANCE_ID}" \
                            --document-name "AWS-RunShellScript" \
                            --comment "Deploy DevSecOps application - build ${BUILD_NUMBER}" \
                            --parameters 'commands=[
                                "docker pull '${FULL_IMAGE}'",
                                "docker stop devsecops-app || true",
                                "docker rm devsecops-app || true",
                                "docker run -d --name devsecops-app --restart unless-stopped -p 80:3000 '${FULL_IMAGE}'",
                                "docker ps",
                                "docker images | grep devsecops-sample-app"
                            ]' \
                            --query 'Command.CommandId' \
                            --output text)

                        echo ""
                        echo "SSM Command ID:"
                        echo "${COMMAND_ID}"

                        echo ""
                        echo "Waiting for deployment command to complete..."

                        for i in $(seq 1 30); do

                            STATUS=$(aws ssm get-command-invocation \
                                --command-id "${COMMAND_ID}" \
                                --instance-id "${EC2_INSTANCE_ID}" \
                                --query 'Status' \
                                --output text)

                            echo "Deployment status: ${STATUS}"

                            if [ "${STATUS}" = "Success" ]; then
                                echo ""
                                echo "EC2 deployment completed successfully."
                                break
                            fi

                            if [ "${STATUS}" = "Failed" ] || \
                               [ "${STATUS}" = "Cancelled" ] || \
                               [ "${STATUS}" = "TimedOut" ]; then

                                echo ""
                                echo "EC2 deployment failed."
                                echo "Command output:"
                                
                                aws ssm get-command-invocation \
                                    --command-id "${COMMAND_ID}" \
                                    --instance-id "${EC2_INSTANCE_ID}" \
                                    --query '[StandardOutputContent,StandardErrorContent]' \
                                    --output text

                                exit 1
                            fi

                            sleep 5

                        done

                        FINAL_STATUS=$(aws ssm get-command-invocation \
                            --command-id "${COMMAND_ID}" \
                            --instance-id "${EC2_INSTANCE_ID}" \
                            --query 'Status' \
                            --output text)

                        if [ "${FINAL_STATUS}" != "Success" ]; then
                            echo "Deployment did not complete successfully."
                            exit 1
                        fi
                    '''
                }
            }
        }

        stage('Application Health Check') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "Application Health Check"
                    echo "=========================================="

                    echo "Checking EC2 application..."

                    sleep 5

                    HTTP_STATUS=$(curl \
                        --silent \
                        --output /tmp/devsecops-response.txt \
                        --write-out "%{http_code}" \
                        --max-time 10 \
                        http://18.209.29.219/ || true)

                    echo ""
                    echo "HTTP Status: ${HTTP_STATUS}"

                    echo ""
                    echo "Application Response:"
                    cat /tmp/devsecops-response.txt || true

                    if [ "${HTTP_STATUS}" = "200" ]; then
                        echo ""
                        echo "Application is UP."
                    else
                        echo ""
                        echo "WARNING: Application health check returned HTTP ${HTTP_STATUS}."
                        exit 1
                    fi
                '''
            }
        }
    }

    post {

        success {
            echo '''
==========================================
DevSecOps Pipeline Completed Successfully
==========================================

Application deployed successfully to EC2.

Application URL:
http://18.209.29.219
'''
        }

        failure {
            echo '''
==========================================
DevSecOps Pipeline FAILED
==========================================

Check the failed stage and Jenkins console output.
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
