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
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/Aseemakram19/starbucks-kubernetes.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=starbucks \
                        -Dsonar.projectKey=starbucks"""
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
                        sh 'docker tag starbucks aseemakram19/starbucks:latest'
                        sh 'docker push aseemakram19/starbucks:latest'
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                sh 'trivy image aseemakram19/starbucks:latest > trivyimage.txt'
            }
        }

        stage('Deploy to Local Cluster') {
            steps {
                dir('kubernetes') {
                    withCredentials([file(credentialsId: 'local-kubeconfig', variable: 'KUBECONFIG')]) {
                        script {
                            sh '''
                            echo "Using kubeconfig: $KUBECONFIG"
                            echo "Verifying cluster access..."
                            kubectl cluster-info

                            echo "Deploying application..."
                            kubectl apply -f manifest.yml

                            echo "Verifying deployment..."
                            kubectl get pods
                            kubectl get svc
                            '''
                        }
                    }
                }
            }
        }
    }
}
