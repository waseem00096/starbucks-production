pipeline {
    agent { label 'kube-master' }  // Run on your Jenkins agent

    tools {
        jdk 'jdk-21'            // JDK configured in Jenkins
        nodejs 'node17'         // Node.js configured in Jenkins
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
                    def qg = waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    echo "SonarQube Quality Gate status: ${qg.status}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('TRIVY FS Scan') {
            steps {
                script {
                    sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL . || true'
                    archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh """
                            docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                script {
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withEnv(["KUBECONFIG=${KUBE_CONFIG}"]) {
                    script {
                        sh 'kubectl version --client'
                        sh """
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' kubernetes/manifest.yml
                            kubectl apply -f kubernetes/manifest.yml
                            kubectl rollout status deployment/starbucks || true
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivyfs.txt,trivyimage.txt', allowEmptyArchive: true
            cleanWs()
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
