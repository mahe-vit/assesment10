pipeline {
    agent any
    
    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for the built Docker image')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run deployment unit validation tests')
    }
    
    environment {
        DOCKER_HUB_CRED = 'docker-hub-credentials'
        DOCKER_USER     = 'mahevit' // Change to your actual Docker Hub profile username
        IMAGE_NAME      = 'predictive-maintenance-dashboard'
    }
    
    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }
        
        stage('Parallel Build & Sanity Linting') {
            parallel {
                stage('Docker Compile') {
                    steps {
                        // Using 'bat' for Windows Command Prompt compilation execution
                        bat "docker build -t %DOCKER_USER%/%IMAGE_NAME%:%DOCKER_IMAGE_TAG% ."
                    }
                }
                stage('Workspace Blueprint Lint') {
                    steps {
                        // Verifies file existence using Windows conditional logic
                        bat "if not exist index.html exit 1"
                        bat "if not exist deployment.yaml exit 1"
                    }
                }
            }
        }
        
        stage('Conditional Integration Test') {
            when { expression { return params.RUN_TESTS } }
            steps {
                // Safely clean up any existing container and perform an HTTP status verification loop
                bat "docker rm -f transient-test-container 2>nul || (exit 0)"
                bat "docker run -d --name transient-test-container -p 8888:80 %DOCKER_USER%/%IMAGE_NAME%:%DOCKER_IMAGE_TAG%"
                bat "timeout /t 5 /nobreak"
                bat "curl -f http://localhost:8888"
                bat "docker rm -f transient-test-container"
            }
        }
        
        stage('Registry Ship') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_HUB_CRED, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    // Authenticates and pushes image using Windows CLI environment parameters
                    bat "echo %PASS% | docker login -u %USER% --password-stdin"
                    bat "docker push %DOCKER_USER%/%IMAGE_NAME%:%DOCKER_IMAGE_TAG%"
                }
            }
        }
        
        stage('Kubernetes Orchestrated Deployment') {
            steps {
                // Modifies the target runtime label strings using powershell or simple inline replacement
                bat "powershell -Command \"(Get-Content deployment.yaml) -replace 'yourdockerhubusername/predictive-maintenance-dashboard:latest', '%DOCKER_USER%/%IMAGE_NAME%:%DOCKER_IMAGE_TAG%' | Set-Content deployment.yaml\""
                bat "kubectl apply -f deployment.yaml"
                bat "kubectl rollout status deployment/predictive-maintenance-deployment --timeout=60s"
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'index.html, deployment.yaml', fingerprint: true
            cleanWs()
        }
    }
}
