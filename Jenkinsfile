pipeline {
    agent {
        label 'windows-docker'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        IMAGE_NAME = 'mohammadarifsunasara/devops-cicd-app'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies and Test') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    if (Test-Path ".venv") { Remove-Item -Recurse -Force ".venv" }
                    python -m venv .venv
                    & ".\\.venv\\Scripts\\python.exe" -m pip install --upgrade pip
                    & ".\\.venv\\Scripts\\python.exe" -m pip install -r requirements.txt
                    & ".\\.venv\\Scripts\\python.exe" -m pytest -v
                '''
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    env.IMAGE_TAG = "build-${env.BUILD_NUMBER}"
                }
                powershell '''
                    $ErrorActionPreference = "Continue"
                    docker build -t "${env:IMAGE_NAME}:${env:IMAGE_TAG}" .
                    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    withEnv(["DOCKER_CONFIG=${env.WORKSPACE}\\.docker-ci"]) {
                        powershell '''
                            $ErrorActionPreference = "Continue"
                            try {
                                $u = $env:DOCKERHUB_USER.Trim()
                                $t = $env:DOCKERHUB_TOKEN.Trim()

                                Write-Host "User length: $($u.Length), token length: $($t.Length)"
                                Write-Host "User: [$u]"
                                Write-Host "Token prefix: $($t.Substring(0, [Math]::Min(9, $t.Length)))"
                                $bytes = [Text.Encoding]::UTF8.GetBytes($t)
                                $hash = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash($bytes)).Replace("-","")
                                Write-Host "Token hash: $($hash.Substring(0,8))"

                                $t | docker login --username $u --password-stdin
                                if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                                docker push "${env:IMAGE_NAME}:${env:IMAGE_TAG}"
                                if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                            }
                            finally {
                                docker logout
                            }
                        '''
                    }
                }
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Continue"
                    $image = "${env:IMAGE_NAME}:${env:IMAGE_TAG}"

                    kind load docker-image $image --name devops-lab
                    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                    $yaml = Get-Content "k8s/deployment.yaml" -Raw
                    $yaml = $yaml.Replace("IMAGE_PLACEHOLDER", $image)
                    Set-Content -Path "k8s/deployment-rendered.yaml" -Value $yaml -Encoding utf8

                    kubectl apply -f k8s/deployment-rendered.yaml
                    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                    kubectl apply -f k8s/service.yaml
                    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                    kubectl rollout status deployment/devops-demo --timeout=180s
                    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                    kubectl get pods
                    kubectl get services
                '''
            }
        }

        stage('Application Health Check') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    $healthy = $false

                    for ($i = 1; $i -le 30; $i++) {
                        try {
                            $response = Invoke-RestMethod -Uri "http://localhost:8081/health" -TimeoutSec 3
                            if ($response.status -eq "healthy") {
                                $healthy = $true
                                break
                            }
                        }
                        catch { }
                        Start-Sleep -Seconds 2
                    }

                    if (-not $healthy) { throw "Application health check failed" }
                    Write-Host "Application is healthy!"
                '''
            }
        }
    }

    post {
        success { echo 'CI/CD pipeline completed successfully!' }
        failure { echo 'Pipeline failed. Check Jenkins console output.' }
        always  { echo 'Pipeline execution finished.' }
    }
}
