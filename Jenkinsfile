pipeline {
    agent any

    tools {
        jdk 'jdk-21'            // JDK configured in Jenkins
        nodejs 'node17'      // Node.js configured in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'   // SonarQube scanner
        KUBE_CONFIG = '/var/lib/jenkins/.kube/config'
        IMAGE_NAME = 'waseem09/starbucks'
        IMAGE_TAG = 'latest'
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
                        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > trivyimage.txt"
            }
        }

        stage('Deploy to Kubernetes') {
           steps {
            withEnv(["KUBECONFIG=${KUBE_CONFIG}"]) {
                 sh """
                # Update the manifest with the latest image tag
                 sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' kubernetes/manifest.yml
            
                 # Apply the manifest to Kubernetes
                 kubectl apply -f  kubernetes/manifest.yml
                 """
        }
    }
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
