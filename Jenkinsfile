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
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    env.SHORT_COMMIT = env.GIT_COMMIT.take(7)
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.SHORT_COMMIT}"
                }
            }
        }

        stage('Unit Test') {
            steps {
                bat 'mvn test'
            }
        }

        stage('Build WAR Package') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat """
                echo Building Docker image...
                echo Image: %IMAGE_NAME%:%IMAGE_TAG%

                docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                """
            }
        }
    }

    post {
        always {
            echo "Build Number: ${env.BUILD_NUMBER}"
            echo "Commit ID: ${env.GIT_COMMIT}"
            echo "Image Tag: ${env.IMAGE_TAG}"
        }

        success {
            echo "Pipeline completed successfully"
        }

        failure {
            echo "Pipeline failed"
        }
    }
}