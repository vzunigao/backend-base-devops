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

        stage('Install and Configure kubectl') {
            steps {
                script {
                    // Descargar e instalar kubectl
                    sh '''
                    curl -LO "https://dl.k8s.io/release/v1.27.0/bin/linux/amd64/kubectl"
                    chmod +x ./kubectl
                    sudo mv ./kubectl /usr/local/bin/kubectl
                    '''
                    
                    // Verificar la instalación
                    sh 'kubectl version --client'
                    
                    // Configurar kubectl
                    // Asegúrate de proporcionar el archivo kubeconfig correcto o variables de entorno
                    sh '''
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG_CONTENT" > $KUBECONFIG
                    '''
                }
            }
        }

        stage('Update Kubernetes Deployment') {
            steps {
                script {
                    def imageTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    sh "kubectl set image deployment/backend-base-deployment backend-base=localhost:8082/backend-base:${imageTag}"
                }
            }
        }
    }
}
