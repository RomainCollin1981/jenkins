pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'rcollin1981'
        APP_NAME = 'fastapiapp'
        DOCKER_CREDENTIALS = credentials('DOCKER_HUB_PASS')
        KUBECONFIG = credentials('config')
        MOVIE_SERVICE_IMAGE = "${DOCKER_REGISTRY}/jenkins_devops_exams_movie_service"
        CAST_SERVICE_IMAGE = "${DOCKER_REGISTRY}/jenkins_devops_exams_cast_service"
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
        
        stage('Pull Docker Images') {
            steps {
                script {
                    // Login to Docker Hub
                    sh "echo ${DOCKER_HUB_PASS} | docker login -u ${DOCKER_REGISTRY} --password-stdin"
                    
                    // Pull existing images
                    sh """
                        docker pull ${MOVIE_SERVICE_IMAGE}
                        docker pull ${CAST_SERVICE_IMAGE}
                    """
                }
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-dev ./charts \
                        --namespace dev \
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
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
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
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
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
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
                input message: 'Deploy to production?', ok: 'Deploy'
                
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-prod ./charts \
                        --namespace prod \
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
                        --set environment=prod
                    """
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker logout'
            cleanWs()
        }
    }
} 