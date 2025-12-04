pipeline {
    agent any
    tools {
        jdk 'jdk-21'
        nodejs 'node17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {

        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/starbucks-production.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh """$SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=starbucks \
                            -Dsonar.projectKey=starbucks"""
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }

        stage('Trivy FS Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t waseem09/starbucks:latest .'
                        sh 'docker push waseem09/starbucks:latest'
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps { sh 'trivy image waseem09/starbucks:latest > trivyimage.txt' }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('kubernetes') {
                    script {
                        sh '''
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        kubectl apply -f manifest.yml
                        kubectl rollout status deployment/starbucks-deployment
                        kubectl get pods
                        kubectl get svc
                        '''
                    }
                }
            }
        }
    }
}
