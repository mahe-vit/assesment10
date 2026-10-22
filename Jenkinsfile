pipeline {
    agent any
    
    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for the built Docker image')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run deployment unit validation tests')
    }
    
    environment {
        DOCKER_HUB_CRED = 'docker-hub-credentials'
        DOCKER_USER     = 'mahevit' 
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
                        // Using explicit Groovy string interpolation to safely bypass Windows cmd variable limitations
                        bat "docker build -t ${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG} ."
                    }
                }
                stage('Workspace Blueprint Lint') {
                    steps {
                        bat "if not exist index.html exit 1"
                        bat "if not exist deployment.yaml exit 1"
                    }
                }
            }
        }
        
        stage('Conditional Integration Test') {
            when { expression { return params.RUN_TESTS } }
            steps {
                bat "docker rm -f transient-test-container 2>nul || (exit 0)"
                bat "docker run -d --name transient-test-container -p 8888:80 ${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}"
                bat "timeout /t 5 /nobreak"
                bat "curl -f http://localhost:8888"
                bat "docker rm -f transient-test-container"
            }
        }
        
        stage('Registry Ship') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_HUB_CRED, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    bat "echo %PASS% | docker login -u %USER% --password-stdin"
                    bat "docker push ${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}"
                }
            }
        }
        
        stage('Kubernetes Orchestrated Deployment') {
            steps {
                bat "powershell -Command \"(Get-Content deployment.yaml) -replace 'yourdockerhubusername/predictive-maintenance-dashboard:latest', '${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}' | Set-Content deployment.yaml\""
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
