
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

        // ==========================================
        // STAGE 1: CHECK ENVIRONMENT AND VERIFY DOCKER
        // ==========================================

        stage('Check Environment') {
            steps {
                script {
                    echo '======================================'
                    echo 'Jenkins Pipeline Started Successfully!'
                    echo "AWS Region: ${REGION}"
                    echo "Repository: ${REPO}"
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo '======================================'

                    echo 'Searching for Docker executable...'

                    // Find Docker through PATH.
                    // If not found, check your known installation path.

                    def dockerPath = powershell(
                        returnStdout: true,
                        script: '''
                            $ErrorActionPreference = 'Stop'

                            $docker = Get-Command docker.exe `
                                -ErrorAction SilentlyContinue

                            if ($docker) {
                                $dockerPath = $docker.Source
                            }
                            else {
                                $candidate = 'C:\\Users\\tejas\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

                                if (Test-Path -LiteralPath $candidate -PathType Leaf) {
                                    $dockerPath = $candidate
                                }
                                else {
                                    Write-Output 'ERROR: Docker executable not found.'
                                    Write-Output 'Check Docker installation and Jenkins PATH.'
                                    exit 1
                                }
                            }

                            Write-Output $dockerPath
                        '''
                    ).trim()

                    if (!dockerPath) {
                        error('Docker executable path could not be detected.')
                    }

                    // Save the discovered Docker path for later stages.
                    env.DOCKER_EXE = dockerPath

                    echo "Docker executable found: ${env.DOCKER_EXE}"

                    // Verify Docker CLI version.
                    echo 'Checking Docker CLI version...'

                    bat "\"${env.DOCKER_EXE}\" --version"

                    // Verify Docker Engine connection.
                    echo 'Checking Docker Engine status...'

                    def dockerStatus = bat(
                        returnStatus: true,
                        script: "\"${env.DOCKER_EXE}\" info"
                    )

                    if (dockerStatus != 0) {
                        error(
                            'Docker CLI was found, but Docker Engine is not accessible. ' +
                            'Start Docker Desktop and check Jenkins service permissions.'
                        )
                    }

                    echo 'SUCCESS: Docker CLI is working!'
                    echo 'SUCCESS: Docker Engine is accessible!'
                }
            }
        }

        // ==========================================
        // STAGE 2: BUILD DOCKER IMAGE
        // ==========================================

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker Image...'

                bat "\"${env.DOCKER_EXE}\" build -t \"${IMG}\" ."

                echo "Docker Image Built: ${IMG}"
            }
        }

        // ==========================================
        // STAGE 3: AWS ECR LOGIN
        // ==========================================

        stage('AWS ECR Login') {
            steps {
                echo 'Authenticating with AWS ECR...'

                withCredentials([
                    string(
                        credentialsId: 'aws-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat """
                        @echo off
                        aws ecr get-login-password --region "${REGION}" --no-cli-pager | "${env.DOCKER_EXE}" login --username AWS --password-stdin "${REGISTRY}"
                    """
                }

                echo 'AWS ECR Login Successful!'
            }
        }

        // ==========================================
        // STAGE 4: PUSH DOCKER IMAGE TO ECR
        // ==========================================

        stage('Push Docker Image') {
            steps {
                echo 'Tagging Docker Image...'

                bat "\"${env.DOCKER_EXE}\" tag \"${IMG}\" \"${REMOTE}\""

                echo 'Pushing Docker Image to AWS ECR...'

                bat "\"${env.DOCKER_EXE}\" push \"${REMOTE}\""

                echo "Image pushed successfully: ${REMOTE}"
            }
        }

        // ==========================================
        // STAGE 5: DEPLOY TO EC2
        // ==========================================

        stage('Deploy to EC2') {
            when {
                expression {
                    return env.INSTANCE != 'none'
                }
            }

            steps {
                echo 'Deploying application to EC2...'

                withCredentials([
                    string(
                        credentialsId: 'aws-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat """
                        aws ssm send-command ^
                        --region "${REGION}" ^
                        --instance-ids "${INSTANCE}" ^
                        --document-name "AWS-RunShellScript" ^
                        --parameters file://deploy.json ^
                        --no-cli-pager
                    """
                }

                echo 'EC2 Deployment Command Submitted!'
            }
        }
    }

    // ==========================================
    // POST BUILD ACTIONS
    // ==========================================

    post {
        success {
            echo '======================================'
            echo 'PIPELINE COMPLETED SUCCESSFULLY!'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED!'
            echo 'Check Console Output for details.'
            echo '======================================'
        }

        always {
            echo 'Jenkins Pipeline Execution Finished.'
        }
    }
}
