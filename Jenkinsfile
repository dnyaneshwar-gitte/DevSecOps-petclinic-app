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
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        MVN = 'C:\\ProgramData\\chocolatey\\lib\\maven\\apache-maven-3.9.15\\bin\\mvn.cmd'
        EMAIL_TO = 'dnyaneshwarg535@gmail.com'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Unit Test') {
            steps {
                bat '"%MVN%" test'
            }
        }

        stage('SonarQube SAST') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    bat """
                    "%MVN%" clean verify sonar:sonar ^
                    -Dsonar.projectKey=devsecops-petclinic ^
                    -Dsonar.projectName=devsecops-petclinic ^
                    -Dsonar.qualitygate.wait=true ^
                    -Dsonar.qualitygate.timeout=1200
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                bat """
                "%MVN%" org.owasp:dependency-check-maven:check ^
                -Dformat=HTML ^
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
                bat '"%MVN%" clean package -DskipTests'
            }
        }

        stage('Hadolint Dockerfile Scan') {
            steps {
                bat """
                echo Hadolint Dockerfile Scan Report > hadolint-report.txt
                echo Dockerfile: Dockerfile >> hadolint-report.txt
                echo. >> hadolint-report.txt

                docker run --rm -i hadolint/hadolint < Dockerfile >> hadolint-report.txt

                echo. >> hadolint-report.txt
                echo Scan completed. >> hadolint-report.txt

                type hadolint-report.txt
                exit /b 0
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'hadolint-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat """
                echo Building Docker image...
                docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    bat """
                    docker run --rm ^
                    -v //var/run/docker.sock:/var/run/docker.sock ^
                    aquasec/trivy:latest image ^
                    --no-progress ^
                    --exit-code 1 ^
                    --severity MEDIUM,HIGH,CRITICAL ^
                    --format table ^
                    %IMAGE_NAME%:%IMAGE_TAG%
                    """
                }
            }
        }

        stage('Run PetClinic Container for ZAP') {
            steps {
                bat """
                docker rm -f petclinic-zap-test || exit /b 0
                docker run -d --name petclinic-zap-test -p 8081:8080 %IMAGE_NAME%:%IMAGE_TAG%
                ping 127.0.0.1 -n 41 > nul
                """
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                bat """
                if not exist zap-report mkdir zap-report

                if "%ZAP_SCAN_TYPE%"=="Baseline" (
                    docker run --rm ^
                    -v "%WORKSPACE%\\zap-report":/zap/wrk ^
                    ghcr.io/zaproxy/zaproxy:stable ^
                    zap-baseline.py ^
                    -t http://host.docker.internal:8081/ ^
                    -r zap-report.html
                )

                if "%ZAP_SCAN_TYPE%"=="API" (
                    docker run --rm ^
                    -v "%WORKSPACE%\\zap-report":/zap/wrk ^
                    ghcr.io/zaproxy/zaproxy:stable ^
                    zap-api-scan.py ^
                    -t http://host.docker.internal:8081/ ^
                    -f openapi ^
                    -r zap-report.html
                )

                if "%ZAP_SCAN_TYPE%"=="FULL" (
                    docker run --rm ^
                    -v "%WORKSPACE%\\zap-report":/zap/wrk ^
                    ghcr.io/zaproxy/zaproxy:stable ^
                    zap-full-scan.py ^
                    -t http://host.docker.internal:8081/ ^
                    -r zap-report.html
                )

                exit /b 0
                """
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
                    bat """
                    aws ecr get-login-password --region %AWS_REGION% | docker login --username AWS --password-stdin %ECR_REGISTRY%
                    """
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                bat """
                docker push %IMAGE_NAME%:%IMAGE_TAG%
                docker push %IMAGE_NAME%:latest
                """
            }
        }
    }

    post {
        always {
            bat 'docker rm -f petclinic-zap-test || exit /b 0'

            archiveArtifacts artifacts: 'hadolint-report.txt,zap-report/zap-report.html,target/dependency-check-report.html', allowEmptyArchive: true

            echo "Build Number: ${env.BUILD_NUMBER}"
            echo "Commit ID: ${env.GIT_COMMIT}"
            echo "Image Tag: ${IMAGE_TAG}"
        }

        success {
            echo "Security pipeline completed successfully"
        }

        failure {
            echo "Security pipeline failed"
        }
    }
}