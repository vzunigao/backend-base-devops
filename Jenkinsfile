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
                curl -LO "https://dl.k8s.io/release/v1.21.0/bin/linux/amd64/kubectl"
                chmod +x ./kubectl
                mv ./kubectl /usr/local/bin/kubectl
                '''
            }
        }

        stage('Setup Kubernetes Config') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig-secret', variable: 'KUBECONFIG_FILE')]) {
                        sh 'export KUBECONFIG=${KUBECONFIG_FILE}'
                    }
                }
            }
        }

        stage('Check Kubernetes Connection') {
            steps {
                sh 'kubectl cluster-info'
            }
        }
        
        stage('Update Kubernetes Deployment'){
            steps {
                script {
                    def imageVersion = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    sh "kubectl set image deployment backend-base-deployment backend-base=localhost:8082/backend-base:${imageVersion}"
                }
            }
        }
    }
}
