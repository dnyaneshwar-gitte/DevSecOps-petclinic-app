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
        EMAIL_TO = 'your-email@gmail.com'
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
                    -Dsonar.qualitygate.wait=true
                    """
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                bat """
                "%MVN%" org.owasp:dependency-check-maven:check ^
                -Dformat=HTML ^
                -DfailBuildOnCVSS=7
                """
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
                docker run --rm -i hadolint/hadolint < Dockerfile > hadolint-report.txt
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
                docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                bat """
                if not exist trivy-cache mkdir trivy-cache

                docker run --rm ^
                -v //var/run/docker.sock:/var/run/docker.sock ^
                -v "%WORKSPACE%":/workspace ^
                -v "%WORKSPACE%\\trivy-cache":/root/.cache/trivy ^
                aquasec/trivy:latest image ^
                --timeout 20m ^
                --severity HIGH,CRITICAL ^
                --format table ^
                --output /workspace/trivy-report.txt ^
                %IMAGE_NAME%:%IMAGE_TAG%

                docker run --rm ^
                -v "%WORKSPACE%":/workspace ^
                pandoc/core:latest ^
                /workspace/trivy-report.txt ^
                -o /workspace/trivy-report.pdf

                docker run --rm ^
                -v //var/run/docker.sock:/var/run/docker.sock ^
                -v "%WORKSPACE%\\trivy-cache":/root/.cache/trivy ^
                aquasec/trivy:latest image ^
                --timeout 20m ^
                --severity HIGH,CRITICAL ^
                --exit-code 1 ^
                %IMAGE_NAME%:%IMAGE_TAG%
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.pdf', allowEmptyArchive: true
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
                    -t http://host.docker.internal:8081/petclinic ^
                    -r zap-report.html
                )

                if "%ZAP_SCAN_TYPE%"=="API" (
                    docker run --rm ^
                    -v "%WORKSPACE%\\zap-report":/zap/wrk ^
                    ghcr.io/zaproxy/zaproxy:stable ^
                    zap-api-scan.py ^
                    -t http://host.docker.internal:8081/petclinic ^
                    -f openapi ^
                    -r zap-report.html
                )

                if "%ZAP_SCAN_TYPE%"=="FULL" (
                    docker run --rm ^
                    -v "%WORKSPACE%\\zap-report":/zap/wrk ^
                    ghcr.io/zaproxy/zaproxy:stable ^
                    zap-full-scan.py ^
                    -t http://host.docker.internal:8081/petclinic ^
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

    //     stage('Login to ECR') {
    //         steps {
    //             withCredentials([[
    //                 $class: 'AmazonWebServicesCredentialsBinding',
    //                 credentialsId: 'aws-creds',
    //                 accessKeyVariable: 'AWS_ACCESS_KEY_ID',
    //                 secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
    //             ]]) {
    //                 bat """
    //                 aws ecr get-login-password --region %AWS_REGION% | docker login --username AWS --password-stdin %ECR_REGISTRY%
    //                 """
    //             }
    //         }
    //     }

    //     stage('Docker Push to ECR') {
    //         steps {
    //             bat """
    //             docker push %IMAGE_NAME%:%IMAGE_TAG%
    //             docker push %IMAGE_NAME%:latest
    //             """
    //         }
    //     }
    // }

    post {
        always {
            bat 'docker rm -f petclinic-zap-test || exit /b 0'

            echo "Build Status: ${currentBuild.currentResult}"
            echo "Build Number: ${env.BUILD_NUMBER}"
            echo "Commit ID: ${env.GIT_COMMIT}"
            echo "Build Link: ${env.BUILD_URL}"

            emailext(
                to: "${EMAIL_TO}",
                subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Build Status: ${currentBuild.currentResult}
Build Number: ${env.BUILD_NUMBER}
Commit ID: ${env.GIT_COMMIT}
Build Link: ${env.BUILD_URL}
Triggered By: ${currentBuild.getBuildCauses()[0].shortDescription}
""",
                attachmentsPattern: 'trivy-report.pdf,hadolint-report.txt,zap-report/zap-report.html'
            )
        }

        success {
            slackSend(
                color: 'good',
                message: """
SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}
Build Link: ${env.BUILD_URL}
Triggered By: ${currentBuild.getBuildCauses()[0].shortDescription}
"""
            )
        }

        failure {
            slackSend(
                color: 'danger',
                message: """
FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}
Build Link: ${env.BUILD_URL}
Triggered By: ${currentBuild.getBuildCauses()[0].shortDescription}
"""
            )
        }
    }
}