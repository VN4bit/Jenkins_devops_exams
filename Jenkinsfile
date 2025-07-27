pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_ID = 'vn4bit'
        MOVIE_SERVICE_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_ID}/movie-service"
        CAST_SERVICE_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_ID}/cast-service"
        KUBECONFIG = credentials('config')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.BUILD_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Movie Service Image') {
                    steps {
                        script {
                            dir('movie-service') {
                                def movieImage = docker.build("${MOVIE_SERVICE_IMAGE}:${BUILD_TAG}")
                                env.MOVIE_IMAGE_TAG = BUILD_TAG
                            }
                        }
                    }
                }
                stage('Build Cast Service Image') {
                    steps {
                        script {
                            dir('cast-service') {
                                def castImage = docker.build("${CAST_SERVICE_IMAGE}:${BUILD_TAG}")
                                env.CAST_IMAGE_TAG = BUILD_TAG
                            }
                        }
                    }
                }
            }
        }
        
        stage('Push to Registry') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    // Login to Docker Hub
                    sh 'docker login -u $DOCKER_ID -p $DOCKER_PASS'
                    
                    // Push movie service image
                    sh "docker push ${MOVIE_SERVICE_IMAGE}:${BUILD_TAG}"
                    
                    // Push cast service image
                    sh "docker push ${CAST_SERVICE_IMAGE}:${BUILD_TAG}"
                    
                    // Tag and push latest for master branch
                    if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'main') {
                        sh """
                            docker tag ${MOVIE_SERVICE_IMAGE}:${BUILD_TAG} ${MOVIE_SERVICE_IMAGE}:latest
                            docker tag ${CAST_SERVICE_IMAGE}:${BUILD_TAG} ${CAST_SERVICE_IMAGE}:latest
                            docker push ${MOVIE_SERVICE_IMAGE}:latest
                            docker push ${CAST_SERVICE_IMAGE}:latest
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Dev') {
            when {
                not { branch 'master' }
                not { branch 'main' }
            }
            steps {
                script {
                    sh """
                        # Update values for dev environment  
                        sed -i 's|tag: dev|tag: ${BUILD_TAG}|g' charts/values-dev.yaml
                        
                        # Deploy to dev namespace
                        helm upgrade --install microservice-dev ./charts \\
                            --namespace dev \\
                            --create-namespace \\
                            --values charts/values-dev.yaml \\
                            --wait
                    """
                }
            }
        }
        
        stage('Deploy to QA') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh """
                        # Update values for QA environment
                        sed -i 's|tag: qa|tag: ${BUILD_TAG}|g' charts/values-qa.yaml
                        
                        # Deploy to QA namespace
                        helm upgrade --install microservice-qa ./charts \\
                            --namespace qa \\
                            --create-namespace \\
                            --values charts/values-qa.yaml \\
                            --wait
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh """
                        # Update values for staging environment
                        sed -i 's|tag: staging|tag: ${BUILD_TAG}|g' charts/values-staging.yaml
                        
                        # Deploy to staging namespace
                        helm upgrade --install microservice-staging ./charts \\
                            --namespace staging \\
                            --create-namespace \\
                            --values charts/values-staging.yaml \\
                            --wait
                    """
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                script {
                    // Create an Approval Button with a timeout of 15 minutes.
                    // this requires a manual validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }
                    
                    sh """
                        # Update values for production environment
                        sed -i 's|tag: latest|tag: latest|g' charts/values-prod.yaml
                        
                        # Deploy to production namespace
                        helm upgrade --install microservice-prod ./charts \\
                            --namespace prod \\
                            --create-namespace \\
                            --values charts/values-prod.yaml \\
                            --wait
                    """
                }
            }
        }
        
        stage('Post-Deployment Tests') {
            parallel {
                stage('Health Check Dev') {
                    when {
                        not { branch 'master' }
                        not { branch 'main' }
                    }
                    steps {
                        sh '''
                            echo "Running health checks for Dev environment..."
                            # Add health check scripts here
                            kubectl get pods -n dev
                        '''
                    }
                }
                stage('Health Check QA') {
                    when {
                        branch 'develop'
                    }
                    steps {
                        sh '''
                            echo "Running health checks for QA environment..."
                            kubectl get pods -n qa
                        '''
                    }
                }
                stage('Health Check Staging') {
                    when {
                        branch 'develop'
                    }
                    steps {
                        sh '''
                            echo "Running health checks for Staging environment..."
                            kubectl get pods -n staging
                        '''
                    }
                }
                stage('Health Check Prod') {
                    when {
                        anyOf {
                            branch 'master'
                            branch 'main'
                        }
                    }
                    steps {
                        sh '''
                            echo "Running health checks for Production environment..."
                            kubectl get pods -n prod
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean up Docker images
            sh '''
                docker system prune -f
            '''
        }
        success {
            echo 'Pipeline succeeded!'
            // Add notification logic here (Slack, email, etc.)
        }
        failure {
            echo 'Pipeline failed!'
            // Add notification logic here
        }
        cleanup {
            cleanWs()
        }
    }
}
