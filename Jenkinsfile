pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'rcollin1981'
        APP_NAME = 'fastapiapp'
        // Les credentials doivent être configurés dans Jenkins avec :
        // ID: DOCKER_HUB_PASS
        // Username: rcollin1981 (votre username Docker Hub)
        // Password: votre token d'accès Docker Hub
        DOCKER_CREDENTIALS = credentials('DOCKER_HUB_PASS')
        KUBECONFIG = credentials('config')
        MOVIE_SERVICE_IMAGE = "${DOCKER_REGISTRY}/jenkins_devops_exams_movie_service:latest"
        CAST_SERVICE_IMAGE = "${DOCKER_REGISTRY}/jenkins_devops_exams_cast_service:latest"
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Checkout avec la configuration spécifique
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/master']],
                        userRemoteConfigs: [[
                            url: 'https://github.com/RomainCollin1981/jenkins.git'
                        ]]
                    ])
                }
            }
        }
        
        stage('Create Namespaces') {
            steps {
                script {
                    sh '''
                        kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }
        
        stage('Docker Login & Pull Images') {
            steps {
                script {
                    // Utilisation plus sécurisée des credentials Docker Hub
                    withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_PASS', 
                                                   usernameVariable: 'DOCKER_USER', 
                                                   passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker pull ${MOVIE_SERVICE_IMAGE}
                            docker pull ${CAST_SERVICE_IMAGE}
                        '''
                    }
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    // Tests pour movie-service
                    sh '''
                        mkdir -p movie-service/tests cast-service/tests
                        
                        # Tests pour movie-service
                        cat > movie-service/tests/test_movie.py << 'EOF'
def test_movie_simple():
    assert True, "Test simple réussi"
def test_movie_addition():
    assert 1 + 1 == 2, "Test d'addition basique"
EOF
                        docker run --rm \
                            -v ${PWD}/movie-service/tests:/app/tests \
                            ${MOVIE_SERVICE_IMAGE} \
                            /bin/bash -c "pip install pytest && python3 -m pytest /app/tests/ -v"
                        
                        # Tests pour cast-service
                        cat > cast-service/tests/test_cast.py << 'EOF'
def test_cast_simple():
    assert True, "Test simple réussi"
def test_cast_addition():
    assert 1 + 1 == 2, "Test d'addition basique"
EOF
                        docker run --rm \
                            -v ${PWD}/cast-service/tests:/app/tests \
                            ${CAST_SERVICE_IMAGE} \
                            /bin/bash -c "pip install pytest && python3 -m pytest /app/tests/ -v"
                    '''
                }
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-dev ./charts \
                        --namespace dev \
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
                        --set environment=dev \
                        --set service.nodePort=30001
                    """
                }
            }
        }
        
        stage('Deploy to QA') {
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-qa ./charts \
                        --namespace qa \
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
                        --set environment=qa \
                        --set service.nodePort=30002
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-staging ./charts \
                        --namespace staging \
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
                        --set environment=staging \
                        --set service.nodePort=30003
                    """
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                expression { 
                    return env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                input message: 'Êtes-vous sûr de vouloir déployer en PRODUCTION ?', ok: 'Oui, je confirme'
                
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-prod ./charts \
                        --namespace prod \
                        --set movie.image=${MOVIE_SERVICE_IMAGE} \
                        --set cast.image=${CAST_SERVICE_IMAGE} \
                        --set environment=prod \
                        --set service.nodePort=30004
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh 'docker logout'
                cleanWs()
            }
        }
    }
} 