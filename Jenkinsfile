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
        
        stage('Code Quality'){
            stages{
                stage('SonarQube analysis'){
                    agent{
                        docker{
                            image 'sonarsource/sonar-scanner-cli'                                   
                            args '--network="devops-infra_default"'
                            reuseNode true
                        }
                    }
                    steps{
                        withSonarQubeEnv('sonarqube'){
                            sh 'sonar-scanner'
                        }
                    }
                }

                stage('Quality Gate'){
                    steps{
                        timeout(time: 10, unit: 'SECONDS'){
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
            }
            
        }

        stage('delivery'){
            steps {
                script{
                    docker.withRegistry('http://localhost:8082', 'nexus-key'){
                        sh 'docker build -t backend-base:latest .'
                        sh "docker tag backend-base:latest localhost:8082/backend-base:latest"
                        sh "docker tag backend-base:latest localhost:8082/backend-base:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                        sh 'docker push localhost:8082/backend-base:latest'
                        sh "docker push localhost:8082/backend-base:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    }
                }
                
            }
        }

        stage('deploy'){
            steps {
                script{
                    docker.withRegistry('http://localhost:8082', 'nexus-key'){
                        sh "docker compose pull"
                        sh "docker compose up --force-recreate --build -d"
                    }
                }
                
            }
        }

        stage('Install kubectl') {
            steps {
                sh '''
                # Instalar kubectl si no está instalado
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/kubectl
                '''
            }
        }

        stage('Authenticate kubectl') {
            steps {
                withCredentials([file(credentialsId: 'kubernetes-key', variable: 'KUBECONFIG')]) {
                    // Se usa el archivo kubeconfig para la autenticación
                    sh 'kubectl config use-context docker-desktop'
                }
            }
        }

        stage('Update Kubernetes deployment') {
            steps {
                script {            
                    def imageName = "localhost:8082/backend-base:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    sh "kubectl set image deployment backend-base-deployment backend-base=${imageName}"
                }
            }
        }


        

        
    }
}
