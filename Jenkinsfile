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
            steps { git branch: 'main', credentialsId: 'github-token', url: 'YOUR_GIT_REPO' }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=starbucks -Dsonar.projectName=starbucks"
                }
            }
        }

        stage('Quality Gate') {
            steps { waitForQualityGate abortPipeline: true }
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
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            emailext (
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Check pipeline logs at ${env.BUILD_URL}",
                to: "your-email@example.com"
            )
        }
    }
}
