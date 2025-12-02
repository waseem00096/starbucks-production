pipeline {
    agent any

    tools {
        jdk 'jdk'            // JDK configured in Jenkins
        nodejs 'node17'      // Node.js configured in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'   // SonarQube scanner
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', 
                    credentialsId: 'github-token', 
                    url: 'https://github.com/waseem00096/starbucks-production.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=starbucks \
                        -Dsonar.projectKey=starbucks
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('TRIVY FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t starbucks .'
                        sh 'docker tag starbucks waseem09/starbucks:latest'
                        sh 'docker push waseem09/starbucks:latest'
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                sh 'trivy image waseem09/starbucks:latest > trivyimage.txt'
            }
        }

        stage('App Deploy to Docker Container') {
            steps {
                sh '''
        # Stop container if already running (ignore errors)
        docker stop starbucks || true

        # Remove old container (ignore errors)
        docker rm starbucks || true

        # Run a fresh container
        docker run -d --name starbucks -p 3000:3000 waseem09/starbucks:latest
        '''
    }
}            }
        }
    }

    post {
        always {
            echo 'Pipeline finished!'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
