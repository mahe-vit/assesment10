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
                        sh "docker build -t ${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG} ."
                    }
                }
                stage('Workspace Blueprint Lint') {
                    steps {
                        sh "test -f index.html"
                        sh "test -f deployment.yaml"
                    }
                }
            }
        }
        
        stage('Conditional Integration Test') {
            when { expression { return params.RUN_TESTS } }
            steps {
                // Ensure no conflicting container exists before running the integration test
                sh "docker rm -f transient-test-container || true"
                sh "docker run -d --name transient-test-container -p 8888:80 ${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}"
                sh "sleep 5"
                sh "curl -f http://localhost:8888"
                sh "docker rm -f transient-test-container"
            }
        }
        
        stage('Registry Ship') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_HUB_CRED, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo '$PASS' | docker login -u '$USER' --password-stdin"
                    sh "docker push ${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}"
                }
            }
        }
        
        stage('Kubernetes Orchestrated Deployment') {
            steps {
                // We use standard shell interaction assuming Jenkins runs on the host machine where kubectl is configured
                sh "sed -i 's|yourdockerhubusername/predictive-maintenance-dashboard:latest|${env.DOCKER_USER}/${env.IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}|g' deployment.yaml"
                sh "kubectl apply -f deployment.yaml"
                sh "kubectl rollout status deployment/predictive-maintenance-deployment --timeout=60s"
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
