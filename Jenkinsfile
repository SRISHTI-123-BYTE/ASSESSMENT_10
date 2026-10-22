
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
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'

                echo 'Docker image built successfully.'
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

                        $docker = "C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe"

                        if (-not (Test-Path $docker)) {
                            throw "Docker executable was not found at the configured path."
                        }

                        $start = New-Object System.Diagnostics.ProcessStartInfo
                        $start.FileName = $docker
                        $start.Arguments = "login --username $env:DOCKER_USER --password-stdin"
                        $start.UseShellExecute = $false
                        $start.RedirectStandardInput = $true
                        $start.RedirectStandardOutput = $true
                        $start.RedirectStandardError = $true
                        $start.CreateNoWindow = $true

                        $process = New-Object System.Diagnostics.Process
                        $process.StartInfo = $start

                        [void]$process.Start()

                        $stdoutTask = $process.StandardOutput.ReadToEndAsync()
                        $stderrTask = $process.StandardError.ReadToEndAsync()

                        $process.StandardInput.WriteLine($env:DOCKER_PASS)
                        $process.StandardInput.Close()

                        $process.WaitForExit()

                        $stdout = $stdoutTask.Result
                        $stderr = $stderrTask.Result

                        if ($stdout) {
                            Write-Output $stdout
                        }

                        if ($process.ExitCode -ne 0) {
                            if ($stderr) {
                                Write-Output $stderr
                            }

                            throw "Docker Hub login failed. Verify the Jenkins credential and token."
                        }

                        Write-Output "Docker Hub login successful."
                    '''

                    bat 'docker push %IMAGE_NAME%:%IMAGE_TAG%'

                    echo 'Docker image pushed to Docker Hub.'
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
                            throw "Failed to apply Kubernetes manifests."
                        }

                        kubectl set image deployment/$env:DEPLOYMENT road-hazard-portal="$env:IMAGE_NAME`:$env:IMAGE_TAG" -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to update the deployment image."
                        }

                        if ($env:ROLLBACK_DEMO -eq "true") {
                            Write-Output "Starting rollback demonstration."

                            kubectl set image deployment/$env:DEPLOYMENT road-hazard-portal="$env:IMAGE_NAME`:does-not-exist" -n $env:K8S_NAMESPACE
                            if ($LASTEXITCODE -ne 0) {
                                throw "Could not set the test image for rollback."
                            }

                            Start-Sleep -Seconds 15

                            kubectl rollout undo deployment/$env:DEPLOYMENT -n $env:K8S_NAMESPACE
                            if ($LASTEXITCODE -ne 0) {
                                throw "Kubernetes rollback failed."
                            }
                        }

                        kubectl rollout status deployment/$env:DEPLOYMENT --timeout=180s -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment rollout did not complete successfully."
                        }

                        Write-Output "Deployment status:"
                        kubectl get deployments -n $env:K8S_NAMESPACE
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not retrieve deployments."
                        }

                        Write-Output "Pod status:"
                        kubectl get pods -l app=road-hazard-portal -n $env:K8S_NAMESPACE

                        Write-Output "Service status:"
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
