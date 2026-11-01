
pipeline {
    agent any

    environment {
        REGION   = 'ap-south-1'
        ACCOUNT  = '123456789012'
        REPO     = 'hello-jenkins'
        INSTANCE = 'none'

        REGISTRY = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG      = "${REPO}:${BUILD_NUMBER}"
        REMOTE   = "${REGISTRY}/${REPO}:latest"
    }

    stages {

        stage('Check Environment') {
            steps {
                echo 'Jenkins Pipeline Started Successfully!'
                echo "AWS Region: ${REGION}"
                echo "Repository: ${REPO}"
                echo "Build Number: ${BUILD_NUMBER}"

                bat 'docker --version'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    bat "docker build -t ${IMG} ."
                }
            }
        }

        stage('AWS ECR Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-key-id',
                           variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret',
                           variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    bat """
                        aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${REGISTRY}
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    bat "docker tag ${IMG} ${REMOTE}"
                    bat "docker push ${REMOTE}"
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression {
                    return env.INSTANCE != 'none'
                }
            }

            steps {
                withCredentials([
                    string(credentialsId: 'aws-key-id',
                           variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret',
                           variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    bat """
                        aws ssm send-command --region ${REGION} --instance-ids ${INSTANCE} --document-name AWS-RunShellScript --parameters file://deploy.json
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline Completed Successfully!'
        }

        failure {
            echo 'Pipeline Failed! Check Console Output.'
        }

        always {
            echo 'Jenkins Pipeline Execution Finished.'
        }
    }
}
