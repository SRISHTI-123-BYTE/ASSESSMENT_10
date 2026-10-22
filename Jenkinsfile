
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

                echo 'All required files are present.'
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

                        if ([string]::IsNullOrWhiteSpace($env:DOCKER_USER)) {
                            throw "Docker Hub username is empty."
                        }

                        if ([string]::IsNullOrEmpty($env:DOCKER_PASS)) {
                            throw "Docker Hub token is empty."
                        }

                        $docker = (Get-Command docker.exe).Source

                        $psi = New-Object System.Diagnostics.ProcessStartInfo
                        $psi.FileName = $docker
                        $psi.Arguments = "login --username `"$env:DOCKER_USER`" --password-stdin"
                        $psi.UseShellExecute = $false
                        $psi.RedirectStandardInput = $true
                        $psi.RedirectStandardOutput = $true
                        $psi.RedirectStandardError = $true
                        $psi.CreateNoWindow = $true

                        $process = New-Object System.Diagnostics.Process
                        $process.StartInfo = $psi
                        [void]$process.Start()

                        $bytes = [System.Text.Encoding]::UTF8.GetBytes(
                            $env:DOCKER_PASS + "`n"
                        )

                        $process.StandardInput.BaseStream.Write(
                            $bytes, 0, $bytes.Length
                        )
                        $process.StandardInput.Close()

                        $stdout = $process.StandardOutput.ReadToEnd()
                        $stderr = $process.StandardError.ReadToEnd()
                        $process.WaitForExit()

                        if ($stdout) {
                            Write-Output $stdout
                        }

                        if ($process.ExitCode -ne 0) {
                            Write-Error "Docker Hub login failed. Verify the saved token and account."
                            exit 1
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
                            throw "Cannot access Kubernetes context."
                        }

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to apply Kubernetes manifests."
                        }

                        kubectl set image deployment/road-hazard-portal road-hazard-portal="${env:IMAGE_NAME}:${env:IMAGE_TAG}" -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to update deployment image."
                        }

                        if ("${env:ROLLBACK_DEMO}" -eq "true") {
                            Write-Output "Starting rollback demonstration."

                            kubectl set image deployment/road-hazard-portal road-hazard-portal="${env:IMAGE_NAME}:does-not-exist" -n $env:K8S_NAMESPACE
                            if ($LASTEXITCODE -ne 0) {
                                throw "Failed to set test image."
                            }

                            Start-Sleep -Seconds 15

                            kubectl rollout undo deployment/road-hazard-portal -n $env:K8S_NAMESPACE
                            if ($LASTEXITCODE -ne 0) {
                                throw "Rollback failed."
                            }
                        }

                        kubectl rollout status deployment/road-hazard-portal --timeout=180s -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment rollout failed."
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
            echo 'Pipeline failed. Review the first error in Console Output.'
        }

        always {
            archiveArtifacts(
                artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                allowEmptyArchive: true
            )
        }
    }
}
