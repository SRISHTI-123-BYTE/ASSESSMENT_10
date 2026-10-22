
pipeline {
    agent any

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: '1',
            description: 'Docker image version'
        )

        booleanParam(
            name: 'ROLLBACK_DEMO',
            defaultValue: false,
            description: 'Demonstrate Kubernetes rollback'
        )
    }

    environment {
        PATH = "C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin;${env.PATH}"
        IMAGE_NAME = 'hsri123/road-hazard-portal'
        DEPLOYMENT = 'road-hazard-portal'
        K8S_NAMESPACE = 'default'
    }

    stages {
        stage('Checkout and Validate') {
            steps {
                checkout scm

                bat 'if not exist index.html exit /b 1'
                bat 'if not exist Dockerfile exit /b 1'
                bat 'if not exist deployment.yaml exit /b 1'

                echo 'All project files are present.'
            }
        }

        stage('Parallel Checks') {
            parallel {
                stage('Check HTML') {
                    steps {
                        bat 'findstr /i "Road Hazard Reporting Portal" index.html'
                    }
                }

                stage('Check Dockerfile') {
                    steps {
                        bat 'findstr /i "nginx" Dockerfile'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker --version'
                bat "docker build -t ${env.IMAGE_NAME}:${params.IMAGE_TAG} ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-pat-final',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        Write-Output "Checking Docker Hub credential binding..."
                        Write-Output "Username received: $env:DOCKER_USER"

                        if ([string]::IsNullOrWhiteSpace($env:DOCKER_USER)) {
                            throw "Docker Hub username is empty."
                        }

                        if ([string]::IsNullOrEmpty($env:DOCKER_PASS)) {
                            throw "Docker Hub token is empty."
                        }

                        Write-Output "Token received by Jenkins."
                        Write-Output "Attempting Docker Hub login..."

                        $env:DOCKER_PASS |
                            docker login --username $env:DOCKER_USER --password-stdin

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub login failed. Verify the saved credential and token."
                        }

                        Write-Output "Docker Hub login succeeded."
                    '''

                    bat "docker push ${env.IMAGE_NAME}:${params.IMAGE_TAG}"
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        kubectl config current-context
                        if ($LASTEXITCODE -ne 0) {
                            throw "Cannot access the Kubernetes context."
                        }

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes manifest application failed."
                        }

                        kubectl set image deployment/road-hazard-portal road-hazard-portal="${env:IMAGE_NAME}:${env:IMAGE_TAG}" -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not update the deployment image."
                        }

                        if ("${env:ROLLBACK_DEMO}" -eq "true") {
                            Write-Output "Starting rollback demonstration..."

                            kubectl set image deployment/road-hazard-portal road-hazard-portal="${env:IMAGE_NAME}:does-not-exist" -n $env:K8S_NAMESPACE
                            if ($LASTEXITCODE -ne 0) {
                                throw "Could not set the test image."
                            }

                            Start-Sleep -Seconds 15

                            kubectl rollout undo deployment/road-hazard-portal -n $env:K8S_NAMESPACE
                            if ($LASTEXITCODE -ne 0) {
                                throw "Kubernetes rollback command failed."
                            }
                        }

                        kubectl rollout status deployment/road-hazard-portal --timeout=180s -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment rollout did not complete successfully."
                        }

                        kubectl get deployments -n $env:K8S_NAMESPACE
                        kubectl get pods -l app=road-hazard-portal -n $env:K8S_NAMESPACE
                        kubectl get services road-hazard-portal-service -n $env:K8S_NAMESPACE
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the first error in Console Output.'
        }

        always {
            archiveArtifacts(
                artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                allowEmptyArchive: true
            )
        }
    }
}
