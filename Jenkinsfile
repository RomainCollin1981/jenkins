pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'rcollin1981'
        APP_NAME = 'fastapiapp'
        // Utiliser les credentials de type 'Username with password'
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        // Utilisation de la variable Jenkins 'config' pour kubeconfig
        KUBECONFIG = credentials('config')
    }
    
    stages {
        stage('Create Namespaces') {
            steps {
                script {
                    // Création des namespaces pour chaque environnement
                    sh '''
                        kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }
        
        stage('Build & Push Docker Images') {
            steps {
                script {
                    // Utilisation sécurisée des credentials Docker
                    withDockerRegistry([ credentialsId: "docker-hub-credentials", url: "" ]) {
                        // Build and push movie service
                        dir('movie-service') {
                            sh """
                                docker build -t ${DOCKER_REGISTRY}/movie-service:${BUILD_NUMBER} .
                                docker push ${DOCKER_REGISTRY}/movie-service:${BUILD_NUMBER}
                            """
                        }
                        
                        // Build and push cast service
                        dir('cast-service') {
                            sh """
                                docker build -t ${DOCKER_REGISTRY}/cast-service:${BUILD_NUMBER} .
                                docker push ${DOCKER_REGISTRY}/cast-service:${BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }
        
        // ... Le reste du Jenkinsfile reste identique ...
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-dev ./charts \
                        --namespace dev \
                        --set image.tag=${BUILD_NUMBER} \
                        --set environment=dev
                    """
                }
            }
        }
        
        stage('Deploy to QA') {
            when {
                branch 'qa'
            }
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-qa ./charts \
                        --namespace qa \
                        --set image.tag=${BUILD_NUMBER} \
                        --set environment=qa
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'staging'
            }
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-staging ./charts \
                        --namespace staging \
                        --set image.tag=${BUILD_NUMBER} \
                        --set environment=staging
                    """
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'master'
            }
            steps {
                // Production deployment requires manual approval
                input message: 'Deploy to production?', ok: 'Deploy'
                
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-prod ./charts \
                        --namespace prod \
                        --set image.tag=${BUILD_NUMBER} \
                        --set environment=prod
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
} 