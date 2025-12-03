pipeline {
    agent { label 'kube-master' }  // Run on master node

    tools {
        jdk 'jdk-21'
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        KUBE_CONFIG = '/var/lib/jenkins/.kube/config'
        IMAGE_NAME = 'waseem09/starbucks'
        IMAGE_TAG = 'latest'
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
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
                    def qg = waitForQualityGate abortPipeline: true, credentialsId: 'Sonar-token'
                    echo "Quality Gate status: ${qg.status}"
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
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL . || true'
                sh 'trivy fs . > trivyfs.txt || true'
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
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > trivyimage.txt || true"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withEnv(["KUBECONFIG=${KUBE_CONFIG}"]) {
                    sh """
                        sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' kubernetes/manifest.yml
                        kubectl apply -f kubernetes/manifest.yml
                        kubectl rollout status deployment/starbucks || true
                    """
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivyfs.txt,trivyimage.txt', allowEmptyArchive: true
            cleanWs()
            echo 'Pipeline Finished!'
        }
        success {
            echo 'Pipeline Completed Successfully!'
        }
        failure {
            echo 'Pipeline Failed!'
        }
    }
}
