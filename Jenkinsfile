pipeline {
    agent any

    environment {
        // .NET Project
        PROJECT_PATH = "PipeLiineJenkinExample\\PipeLiineJenkinExample.csproj"
        PUBLISH_FOLDER = "publish"

        // Target Windows EC2 Private IP
        TARGET_SERVER = "184.169.246.116"

        // IIS Configuration
        IIS_APP_POOL = "MyMvcAppPool"
        IIS_SITE_NAME = "MyMvcApp"

        // Deployment folder on target EC2
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
                    dotnet test --configuration Release --no-build
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

                    $password = ConvertTo-SecureString `
                        $env:WIN_PASSWORD `
                        -AsPlainText `
                        -Force

                    $credential = New-Object `
                        System.Management.Automation.PSCredential `
                        ($username, $password)

                    Write-Host "Connecting to $server"

                    # Create Remote PowerShell Session
                    $session = New-PSSession `
                        -ComputerName $server `
                        -Credential $credential

                    # Stop IIS Application Pool
                    Write-Host "Stopping IIS Application Pool"

                    Invoke-Command -Session $session -ScriptBlock {

                        Import-Module WebAdministration

                        $appPool = "MyMvcAppPool"

                        if (Test-Path "IIS:\\AppPools\\$appPool") {

                            Stop-WebAppPool -Name $appPool

                            Write-Host "Application Pool Stopped"
                        }

                    }

                    # Clean deployment directory
                    Write-Host "Cleaning old deployment files"

                    Invoke-Command -Session $session -ScriptBlock {

                        $deployPath = "C:\\inetpub\\wwwroot\\MyMvcApp"

                        if (Test-Path $deployPath) {

                            Get-ChildItem $deployPath -Force |
                            Remove-Item -Recurse -Force

                        }
                        else {

                            New-Item `
                                -ItemType Directory `
                                -Path $deployPath `
                                -Force
                        }

                    }

                    # Copy published application
                    Write-Host "Copying published files"

                    Copy-Item `
                        -Path "$env:WORKSPACE\\publish\\*" `
                        -Destination "C:\\inetpub\\wwwroot\\MyMvcApp" `
                        -ToSession $session `
                        -Recurse `
                        -Force

                    # Start IIS Application Pool
                    Write-Host "Starting IIS Application Pool"

                    Invoke-Command -Session $session -ScriptBlock {

                        Import-Module WebAdministration

                        $appPool = "MyMvcAppPool"

                        Start-WebAppPool -Name $appPool

                        Write-Host "Application Pool Started"

                    }

                    # Close Remote Session
                    Remove-PSSession $session

                    Write-Host "Deployment Completed Successfully!"

                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'Deployment completed successfully!'
        }

        failure {
            echo 'Deployment failed!'
        }

        always {
            cleanWs()
        }
    }
}