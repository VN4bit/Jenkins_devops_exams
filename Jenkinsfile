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
                    // Get current branch name more reliably
                    env.BRANCH_NAME = env.BRANCH_NAME ?: sh(
                        script: "git rev-parse --abbrev-ref HEAD",
                        returnStdout: true
                    ).trim()
                    
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    
                    // Ensure we have valid values, fallback to defaults if needed
                    def branchName = env.BRANCH_NAME ?: 'main'
                    def buildNumber = env.BUILD_NUMBER ?: '1'
                    def commitShort = env.GIT_COMMIT_SHORT ?: 'unknown'
                    
                    env.BUILD_TAG = "${branchName}-${buildNumber}-${commitShort}"
                    
                    echo "Branch: ${branchName}"
                    echo "Build Number: ${buildNumber}"  
                    echo "Commit: ${commitShort}"
                    echo "Build Tag: ${env.BUILD_TAG}"
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Movie Service Image') {
                    steps {
                        script {
                            dir('movie-service') {
                                def movieImage = docker.build("${MOVIE_SERVICE_IMAGE}:${BUILD_TAG}", "--no-cache .")
                                env.MOVIE_IMAGE_TAG = BUILD_TAG
                            }
                        }
                    }
                }
                stage('Build Cast Service Image') {
                    steps {
                        script {
                            dir('cast-service') {
                                def castImage = docker.build("${CAST_SERVICE_IMAGE}:${BUILD_TAG}", "--no-cache .")
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
                        
                        # Only clean up the target namespace (dev) that might conflict
                        TARGET_NS="dev"
                        if kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                            echo "Checking namespace \$TARGET_NS..."
                            if ! kubectl get namespace \$TARGET_NS -o jsonpath='{.metadata.labels.app\\.kubernetes\\.io/managed-by}' | grep -q "Helm"; then
                                echo "Namespace \$TARGET_NS exists but not managed by Helm, deleting..."
                                kubectl delete namespace \$TARGET_NS --ignore-not-found=true
                                
                                # Wait for target namespace to be fully deleted with timeout
                                echo "Waiting for namespace \$TARGET_NS to be deleted..."
                                for i in {1..60}; do
                                    if ! kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                                        echo "Namespace \$TARGET_NS successfully deleted."
                                        break
                                    fi
                                    if [ \$i -eq 60 ]; then
                                        echo "Warning: Namespace deletion timed out after 2 minutes, proceeding anyway..."
                                        break
                                    fi
                                    sleep 2
                                done
                            else
                                echo "Namespace \$TARGET_NS is already managed by Helm, skipping deletion."
                            fi
                        else
                            echo "Namespace \$TARGET_NS does not exist, proceeding with deployment."
                        fi
                        
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
                        
                        # Only clean up the target namespace (qa) that might conflict
                        TARGET_NS="qa"
                        if kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                            echo "Checking namespace \$TARGET_NS..."
                            if ! kubectl get namespace \$TARGET_NS -o jsonpath='{.metadata.labels.app\\.kubernetes\\.io/managed-by}' | grep -q "Helm"; then
                                echo "Namespace \$TARGET_NS exists but not managed by Helm, deleting..."
                                kubectl delete namespace \$TARGET_NS --ignore-not-found=true
                                
                                # Wait for target namespace to be fully deleted with timeout
                                echo "Waiting for namespace \$TARGET_NS to be deleted..."
                                for i in {1..60}; do
                                    if ! kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                                        echo "Namespace \$TARGET_NS successfully deleted."
                                        break
                                    fi
                                    if [ \$i -eq 60 ]; then
                                        echo "Warning: Namespace deletion timed out after 2 minutes, proceeding anyway..."
                                        break
                                    fi
                                    sleep 2
                                done
                            else
                                echo "Namespace \$TARGET_NS is already managed by Helm, skipping deletion."
                            fi
                        else
                            echo "Namespace \$TARGET_NS does not exist, proceeding with deployment."
                        fi
                        
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
                        
                        # Only clean up the target namespace (staging) that might conflict
                        TARGET_NS="staging"
                        if kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                            echo "Checking namespace \$TARGET_NS..."
                            if ! kubectl get namespace \$TARGET_NS -o jsonpath='{.metadata.labels.app\\.kubernetes\\.io/managed-by}' | grep -q "Helm"; then
                                echo "Namespace \$TARGET_NS exists but not managed by Helm, deleting..."
                                kubectl delete namespace \$TARGET_NS --ignore-not-found=true
                                
                                # Wait for target namespace to be fully deleted with timeout
                                echo "Waiting for namespace \$TARGET_NS to be deleted..."
                                for i in {1..60}; do
                                    if ! kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                                        echo "Namespace \$TARGET_NS successfully deleted."
                                        break
                                    fi
                                    if [ \$i -eq 60 ]; then
                                        echo "Warning: Namespace deletion timed out after 2 minutes, proceeding anyway..."
                                        break
                                    fi
                                    sleep 2
                                done
                            else
                                echo "Namespace \$TARGET_NS is already managed by Helm, skipping deletion."
                            fi
                        else
                            echo "Namespace \$TARGET_NS does not exist, proceeding with deployment."
                        fi
                        
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
                        
                        # Only clean up the target namespace (prod) that might conflict
                        TARGET_NS="prod"
                        if kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                            echo "Checking namespace \$TARGET_NS..."
                            if ! kubectl get namespace \$TARGET_NS -o jsonpath='{.metadata.labels.app\\.kubernetes\\.io/managed-by}' | grep -q "Helm"; then
                                echo "Namespace \$TARGET_NS exists but not managed by Helm, deleting..."
                                kubectl delete namespace \$TARGET_NS --ignore-not-found=true
                                
                                # Wait for target namespace to be fully deleted with timeout
                                echo "Waiting for namespace \$TARGET_NS to be deleted..."
                                for i in {1..60}; do
                                    if ! kubectl get namespace \$TARGET_NS >/dev/null 2>&1; then
                                        echo "Namespace \$TARGET_NS successfully deleted."
                                        break
                                    fi
                                    if [ \$i -eq 60 ]; then
                                        echo "Warning: Namespace deletion timed out after 2 minutes, proceeding anyway..."
                                        break
                                    fi
                                    sleep 2
                                done
                            else
                                echo "Namespace \$TARGET_NS is already managed by Helm, skipping deletion."
                            fi
                        else
                            echo "Namespace \$TARGET_NS does not exist, proceeding with deployment."
                        fi
                        
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
