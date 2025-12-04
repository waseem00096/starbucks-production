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
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/starbucks-production.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    try {
                        withSonarQubeEnv('SonarQube') {
                            sh """$SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectName=starbucks \
                                -Dsonar.projectKey=starbucks"""
                        }
                    } catch (err) {
                        echo "SonarQube analysis failed, continuing pipeline..."
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    try {
                        waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    } catch (err) {
                        echo "Quality gate failed, continuing pipeline..."
                    }
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

        stage('Deploy to Local Cluster') {
            steps {
                dir('kubernetes') {
                    script {
                        sh '''
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
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
    } // end of stages
} // end of pipeline
