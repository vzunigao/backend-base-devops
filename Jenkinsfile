pipeline {
    agent any
    stages {
        stage('Build and test') {
            agent {
                docker {
                    image 'node:20.11.1-alpine3.19' 
                    reuseNode true
                }
            }
            stages {
                stage('Instalar dependencias') {
                    steps {
                        sh 'npm install'
                    }
                }
                stage('ejecucion de test') {
                    steps {
                        sh 'npm run test'
                    }
                }
                stage('ejecucion de build') {
                    steps {
                        sh 'npm run build'
                    }
                }
            }
        }

        stage('Code Quality') {
            stages {
                stage('SonarQube analysis') {
                    agent {
                        docker {
                            image 'sonarsource/sonar-scanner-cli'
                            args '--network="devops-infra_default"'
                            reuseNode true
                        }
                    }
                    steps {
                        withSonarQubeEnv('sonarqube') {
                            sh 'sonar-scanner'
                        }
                    }
                }

                stage('Quality Gate') {
                    steps {
                        timeout(time: 10, unit: 'SECONDS') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
            }
        }

        stage('delivery') {
            steps {
                script {
                    docker.withRegistry('http://localhost:8082', 'nexus-key') {
                        sh 'docker build -t backend-base:latest .'
                        sh "docker tag backend-base:latest localhost:8082/backend-base:latest"
                        sh "docker tag backend-base:latest localhost:8082/backend-base-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                        sh 'docker push localhost:8082/backend-base:latest'
                        sh "docker push localhost:8082/backend-base-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('deploy') {
            steps {
                script {
                    docker.withRegistry('http://localhost:8082', 'nexus-key') {
                        sh "docker compose pull"
                        sh "docker compose up --force-recreate --build -d"
                    }
                }
            }
        }

        stage('Kubernetes Deployment') {
            steps {
                script {
                    // Sanitizar branch name y crear imageName
                    def sanitizedBranchName = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9-]', '-')
                    def imageName = "localhost:8082/backend-base-${sanitizedBranchName}-${env.BUILD_NUMBER}".replace(':', '-')

                    // Imprimir el nombre de la imagen para depuración
                    echo "Deploying image: ${imageName}"

                    withKubeConfig([credentialsId: 'kubeconfig-id']) {
                        sh "kubectl set image deployment backend-base-deployment backend-base=${imageName}"
                    }
                }
            }
        }
    }
}
