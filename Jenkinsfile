 pipeline {
    agent any

    environment {
        PROJECT_PATH = "PipeLiineJenkinExample\\PipeLiineJenkinExample.csproj"
        PUBLISH_FOLDER = "publish"

        TARGET_SERVER = "184.169.246.116"

        IIS_APP_POOL = "MyMvcAppPool"
        IIS_SITE_NAME = "MyMvcApp"

        DEPLOY_PATH = "C:\\inetpub\\wwwroot\\MyMvcApp"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check .NET Version') {
            steps {
                bat '''
                    dotnet --version
                    dotnet --info
                '''
            }
        }

        stage('Restore') {
            steps {
                bat '''
                    dotnet restore "%PROJECT_PATH%"
                '''
            }
        }

        stage('Build') {
            steps {
                bat '''
                    dotnet build "%PROJECT_PATH%" --configuration Release --no-restore
                '''
            }
        }

        stage('Test') {
            steps {
                bat '''
                    dotnet test "%PROJECT_PATH%" --configuration Release --no-build
                '''
            }
        }

        stage('Publish') {
            steps {
                bat '''
                    if exist "%PUBLISH_FOLDER%" rmdir /s /q "%PUBLISH_FOLDER%"

                    dotnet publish "%PROJECT_PATH%" ^
                        --configuration Release ^
                        --output "%PUBLISH_FOLDER%"
                '''
            }
        }

        stage('Deploy to Windows EC2') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'windows-ec2-credentials',
                        usernameVariable: 'WIN_USER',
                        passwordVariable: 'WIN_PASSWORD'
                    )
                ]) {

                    powershell '''

                    $ErrorActionPreference = "Stop"

                    $server = $env:TARGET_SERVER
                    $username = $env:WIN_USER

                    Write-Host "========================================"
                    Write-Host "Connecting to Windows EC2: $server"
                    Write-Host "========================================"

                    $password = ConvertTo-SecureString `
                        $env:WIN_PASSWORD `
                        -AsPlainText `
                        -Force

                    $credential = New-Object `
                        System.Management.Automation.PSCredential(
                            $username,
                            $password
                        )

                    $session = $null

                    try {

                        # Create PowerShell Remoting Session
                        $session = New-PSSession `
                            -ComputerName $server `
                            -Credential $credential

                        Write-Host "Connected successfully."

                        # Stop IIS Application Pool
                        Write-Host "Stopping IIS Application Pool..."

                        Invoke-Command `
                            -Session $session `
                            -ScriptBlock {

                                Import-Module WebAdministration

                                $appPool = "MyMvcAppPool"

                                if (Test-Path "IIS:\\AppPools\\$appPool") {

                                    $state = (Get-WebAppPoolState `
                                        -Name $appPool).Value

                                    if ($state -eq "Started") {

                                        Stop-WebAppPool `
                                            -Name $appPool

                                        Write-Host "Application Pool stopped."

                                    }
                                    else {

                                        Write-Host "Application Pool already stopped."

                                    }

                                }
                                else {

                                    throw "IIS Application Pool '$appPool' does not exist."

                                }
                            }

                        # Create Deployment Directory
                        Write-Host "Preparing deployment directory..."

                        Invoke-Command `
                            -Session $session `
                            -ScriptBlock {

                                $deployPath = "C:\\inetpub\\wwwroot\\MyMvcApp"

                                if (-not (Test-Path $deployPath)) {

                                    New-Item `
                                        -ItemType Directory `
                                        -Path $deployPath `
                                        -Force | Out-Null

                                    Write-Host "Deployment directory created."

                                }
                            }

                        # Remove Old Application Files
                        Write-Host "Removing old application files..."

                        Invoke-Command `
                            -Session $session `
                            -ScriptBlock {

                                $deployPath = "C:\\inetpub\\wwwroot\\MyMvcApp"

                                Get-ChildItem `
                                    -Path $deployPath `
                                    -Force |
                                Remove-Item `
                                    -Recurse `
                                    -Force

                                Write-Host "Old application files removed."

                            }

                        # Copy Published Application
                        Write-Host "Copying published application..."

                        Copy-Item `
                            -Path "$env:WORKSPACE\\publish\\*" `
                            -Destination "C:\\inetpub\\wwwroot\\MyMvcApp" `
                            -ToSession $session `
                            -Recurse `
                            -Force

                        Write-Host "Application files copied successfully."

                        # Start IIS Application Pool
                        Write-Host "Starting IIS Application Pool..."

                        Invoke-Command `
                            -Session $session `
                            -ScriptBlock {

                                Import-Module WebAdministration

                                $appPool = "MyMvcAppPool"

                                Start-WebAppPool `
                                    -Name $appPool

                                Write-Host "Application Pool started successfully."

                            }

                        Write-Host "========================================"
                        Write-Host "Deployment completed successfully!"
                        Write-Host "========================================"

                    }
                    finally {

                        if ($null -ne $session) {

                            Write-Host "Closing PowerShell session..."

                            Remove-PSSession `
                                -Session $session
                        }
                    }
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'Deployment Successful!'
        }

        failure {
            echo 'Deployment Failed!'
        }

        always {
            cleanWs()
        }
    }
}