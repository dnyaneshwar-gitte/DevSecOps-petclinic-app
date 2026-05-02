pipeline {
    agent any

    parameters {
        choice(
            name: 'ZAP_SCAN_TYPE',
            choices: ['Baseline', 'API', 'FULL'],
            description: 'Select OWASP ZAP scan type'
        )
    }

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO_NAME = 'devsecops-petclinic'
        AWS_ACCOUNT_ID = '038462753889'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
        MVN = 'mvn'
        EMAIL_TO = 'dnyaneshwarg535@gmail.com'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    env.SHORT_COMMIT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${BUILD_NUMBER}-${SHORT_COMMIT}"
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh '${MVN} test'
            }
        }

        stage('SonarQube SAST') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                    ${MVN} clean verify sonar:sonar \
                    -Dsonar.projectKey=devsecops-petclinic \
                    -Dsonar.projectName=devsecops-petclinic \
                    -Dsonar.qualitygate.wait=true \
                    -Dsonar.qualitygate.timeout=1200
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh """
                ${MVN} org.owasp:dependency-check-maven:check \
                -Dformat=HTML \
                -DfailBuildOnCVSS=11
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/dependency-check-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Build WAR Package') {
            steps {
                sh '${MVN} clean package -DskipTests'
            }
        }

        stage('Hadolint Dockerfile Scan') {
            steps {
                sh '''
                echo "Hadolint Dockerfile Scan Report" > hadolint-report.txt
                docker run --rm -i hadolint/hadolint < Dockerfile >> hadolint-report.txt
                cat hadolint-report.txt
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'hadolint-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh """
                    docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy:latest image \
                    --no-progress \
                    --exit-code 1 \
                    --severity MEDIUM,HIGH,CRITICAL \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Run PetClinic Container for ZAP') {
            steps {
                sh """
                docker rm -f petclinic-zap-test || true
                docker run -d --name petclinic-zap-test -p 8081:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                sleep 40
                """
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                sh '''
                mkdir -p zap-report

                if [ "$ZAP_SCAN_TYPE" = "Baseline" ]; then
                    docker run --rm \
                    -v $(pwd)/zap-report:/zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py \
                    -t http://localhost:8081/ \
                    -r zap-report.html
                fi

                if [ "$ZAP_SCAN_TYPE" = "API" ]; then
                    docker run --rm \
                    -v $(pwd)/zap-report:/zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-api-scan.py \
                    -t http://localhost:8081/ \
                    -f openapi \
                    -r zap-report.html
                fi

                if [ "$ZAP_SCAN_TYPE" = "FULL" ]; then
                    docker run --rm \
                    -v $(pwd)/zap-report:/zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-full-scan.py \
                    -t http://localhost:8081/ \
                    -r zap-report.html
                fi
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report/zap-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh """
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                docker push ${IMAGE_NAME}:latest
                """
            }
        }
    }

    post {
        always {
            sh 'docker rm -f petclinic-zap-test || true'

            archiveArtifacts artifacts: 'hadolint-report.txt,zap-report/zap-report.html,target/dependency-check-report.html', allowEmptyArchive: true

            echo "Build Number: ${env.BUILD_NUMBER}"
            echo "Commit ID: ${env.SHORT_COMMIT}"
            echo "Image Tag: ${env.IMAGE_TAG}"
        }

        success {
            echo "Security pipeline completed successfully"
        }

        failure {
            echo "Security pipeline failed"
        }
    }
}