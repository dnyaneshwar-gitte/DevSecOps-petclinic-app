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

        stage('SonarQube Quality Gate') {
            steps {
                script {
                    timeout(time: 20, unit: 'MINUTES') {
                        def qg = waitForQualityGate abortPipeline: false

                        if (qg.status != 'OK') {
                            error "Pipeline failed because SonarQube Quality Gate status is: ${qg.status}"
                        }

                        echo "SonarQube Quality Gate passed: ${qg.status}"
                    }
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
                echo Building Docker image...
                docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                bat """
                docker run --rm ^
                -v //var/run/docker.sock:/var/run/docker.sock ^
                -v "%WORKSPACE%":/workspace ^
                aquasec/trivy:latest image ^
                --severity HIGH,CRITICAL ^
                --format json ^
                --output /workspace/trivy-report.json ^
                %IMAGE_NAME%:%IMAGE_TAG%

                docker run --rm ^
                -v "%WORKSPACE%":/workspace ^
                aquasec/trivy:latest convert ^
                --format table ^
                --output /workspace/trivy-report.txt ^
                /workspace/trivy-report.json

                docker run --rm ^
                -v "%WORKSPACE%":/workspace ^
                pandoc/core:latest ^
                /workspace/trivy-report.txt ^
                -o /workspace/trivy-report.pdf

                docker run --rm ^
                -v //var/run/docker.sock:/var/run/docker.sock ^
                aquasec/trivy:latest image ^
                --severity HIGH,CRITICAL ^
                --exit-code 1 ^
                %IMAGE_NAME%:%IMAGE_TAG%
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json,trivy-report.txt,trivy-report.pdf', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
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