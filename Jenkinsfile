
pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: '1',
               description: 'Docker image version')
        booleanParam(name: 'ROLLBACK_DEMO', defaultValue: false,
                     description: 'Deploy a faulty image to demonstrate automatic rollback')
    }

    environment {
        IMAGE_NAME = 'hsri123/road-hazard-portal'
        DEPLOYMENT = 'road-hazard-portal'
    }

    stages {
        stage('Checkout and Validate') {
            steps {
                checkout scm
                bat 'dir'
                bat 'if not exist index.html exit /b 1'
                bat 'if not exist Dockerfile exit /b 1'
                bat 'if not exist deployment.yaml exit /b 1'
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
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKER_TOKEN% | docker login -u %DOCKER_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %IMAGE_NAME%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig',
                         variable: 'KUBECONFIG_FILE')
                ]) {
                    script {
                        bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" apply -f deployment.yaml
                            if errorlevel 1 exit /b 1

                            kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/%DEPLOYMENT% road-hazard-portal=%IMAGE_NAME%:%IMAGE_TAG%
                            if errorlevel 1 exit /b 1
                        '''

                        if (params.ROLLBACK_DEMO) {
                            bat '''
                                kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/%DEPLOYMENT% road-hazard-portal=%IMAGE_NAME%:does-not-exist
                            '''
                        }

                        def rolloutStatus = bat(
                            script: 'kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT% --timeout=90s',
                            returnStatus: true
                        )

                        if (rolloutStatus != 0) {
                            echo 'Deployment failed. Rolling back to the previous revision.'

                            bat '''
                                kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout undo deployment/%DEPLOYMENT%
                                if errorlevel 1 exit /b 1
                                kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''

                            error('Faulty rollout detected; rollback executed.')
                        }

                        bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" get deployments
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" get services
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                             allowEmptyArchive: true
        }
    }
}
