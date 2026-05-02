pipeline {
    agent { label 'shipquick-slave' }   // ✅ FORCE SLAVE

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

        stage('Verify Tools on Slave') {
            steps {
                sh '''
                echo "Checking tools..."
                java -version
                mvn -v
                docker --version
                aws --version
                '''
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
                    -Dsonar.projectName=devsecops-petclinic
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh '''
                dependency-check.sh \
                  --project "petclinic" \
                  --scan . \
                  --format HTML \
                  --out dependency-check-report \
                  --failOnCVSS 7
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/*', allowEmptyArchive: true
                }
            }
        }

        stage('Build WAR') {
            steps {
                sh '${MVN} clean package -DskipTests'
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

        stage('Trivy Scan') {
            steps {
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Run App for ZAP') {
            steps {
                sh '''
                docker rm -f petclinic-zap-test || true
                docker run -d --name petclinic-zap-test -p 8081:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                sleep 40
                '''
            }
        }

        stage('ZAP Scan') {
            steps {
                sh '''
                mkdir -p zap-report

                if [ "$ZAP_SCAN_TYPE" = "Baseline" ]; then
                    docker run --rm -v $(pwd)/zap-report:/zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py -t http://localhost:8081/ -r zap-report.html
                fi

                if [ "$ZAP_SCAN_TYPE" = "API" ]; then
                    docker run --rm -v $(pwd)/zap-report:/zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-api-scan.py -t http://localhost:8081/ -f openapi -r zap-report.html
                fi

                if [ "$ZAP_SCAN_TYPE" = "FULL" ]; then
                    docker run --rm -v $(pwd)/zap-report:/zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-full-scan.py -t http://localhost:8081/ -r zap-report.html
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
                sh '''
                aws ecr get-login-password --region us-east-1 | \
                docker login --username AWS --password-stdin 038462753889.dkr.ecr.us-east-1.amazonaws.com
                '''
            }
        }

        stage('Push Image') {
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

            echo "Build: ${BUILD_NUMBER}"
            echo "Commit: ${SHORT_COMMIT}"
            echo "Tag: ${IMAGE_TAG}"
        }

        success {
            echo "Pipeline SUCCESS"
        }

        failure {
            echo "Pipeline FAILED"
        }
    }
}