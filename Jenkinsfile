
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
            description: 'Deploy a faulty image to demonstrate automatic rollback'
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

                echo 'All required project files are present.'
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
                        credentialsId: 'dockerhub-pat-new',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $env:DOCKER_PASS |
                            docker login --username $env:DOCKER_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            exit $LASTEXITCODE
                        }
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
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        kubectl set image deployment/road-hazard-portal road-hazard-portal="${env:IMAGE_NAME}:${env:IMAGE_TAG}" -n default
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        if ("${env:ROLLBACK_DEMO}" -eq "true") {
                            kubectl set image deployment/road-hazard-portal road-hazard-portal="${env:IMAGE_NAME}:does-not-exist" -n default
                            if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                            Start-Sleep -Seconds 10
                            kubectl rollout undo deployment/road-hazard-portal -n default
                            if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                            kubectl rollout status deployment/road-hazard-portal --timeout=120s -n default
                            if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                            Write-Output "Rollback demonstration completed."
                        }
                        else {
                            kubectl rollout status deployment/road-hazard-portal --timeout=180s -n default
                            if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                        }

                        kubectl get deployments -n default
                        kubectl get pods -l app=road-hazard-portal -n default
                        kubectl get services road-hazard-portal-service -n default
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
