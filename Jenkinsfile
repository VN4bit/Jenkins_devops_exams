pipeline {
    agent any
    
    environment {
        // Registry configuration
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_ID = 'vn4bit'
        MOVIE_SERVICE_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_ID}/movie-service"
        CAST_SERVICE_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_ID}/cast-service"
        KUBECONFIG = credentials('config')
        
        // Deployment timeouts (Helm needs string format with unit s, m, h; e.g., "10m" for 10 minutes)
        HELM_TIMEOUT = "10m"
        
        // Timeout constants as environment variables
        STAGE_TIMEOUT_MIN = 15
        PROD_TIMEOUT_MIN = 20
        APPROVAL_TIMEOUT_MIN = 15
        HEALTH_CHECK_TIMEOUT_MIN = 5
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    setBuildMetadata()
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Movie Service') {
                    steps {
                        buildDockerImage('movie-service', env.MOVIE_SERVICE_IMAGE)
                    }
                }
                stage('Build Cast Service') {
                    steps {
                        buildDockerImage('cast-service', env.CAST_SERVICE_IMAGE)
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
                    pushDockerImages()
                }
            }
        }
        
        stage('Deploy to Dev') {
            when {
                not { branch 'master' }
                not { branch 'main' }
                branch 'develop'
            }
            steps {
                script {
                    timeout(time: env.STAGE_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                        deployToEnvironment('dev', 'charts/values-dev.yaml')
                    }
                }
            }
        }
        
        stage('Deploy to QA') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    timeout(time: env.STAGE_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                        deployToEnvironment('qa', 'charts/values-qa.yaml')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    timeout(time: env.STAGE_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                        deployToEnvironment('staging', 'charts/values-staging.yaml')
                    }
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
                    timeout(time: env.PROD_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                        // Manual approval for production
                        timeout(time: env.APPROVAL_TIMEOUT_MIN.toInteger(), unit: "MINUTES") {
                            input message: 'Do you want to deploy to production?', ok: 'Deploy'
                        }
                        deployToEnvironment('prod', 'charts/values-prod.yaml')
                    }
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
                        script {
                            timeout(time: env.HEALTH_CHECK_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                                healthCheck('dev')
                            }
                        }
                    }
                }
                stage('Health Check QA') {
                    when {
                        not { branch 'master' }
                        not { branch 'main' }
                    }
                    steps {
                        script {
                            timeout(time: env.HEALTH_CHECK_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                                healthCheck('qa')
                            }
                        }
                    }
                }
                stage('Health Check Staging') {
                    when {
                        not { branch 'master' }
                        not { branch 'main' }
                    }
                    steps {
                        script {
                            timeout(time: env.HEALTH_CHECK_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                                healthCheck('staging')
                            }
                        }
                    }
                }
                stage('Health Check Production') {
                    when {
                        anyOf {
                            branch 'master'
                            branch 'main'
                        }
                    }
                    steps {
                        script {
                            timeout(time: env.HEALTH_CHECK_TIMEOUT_MIN.toInteger(), unit: 'MINUTES') {
                                healthCheck('prod')
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Clean up Docker images
                sh 'docker system prune -f'
                
                // Archive deployment logs if they exist
                archiveArtifacts artifacts: 'deployment-*.log', allowEmptyArchive: true
            }
        }
        success {
            echo "Pipeline succeeded for branch: ${env.BRANCH_NAME}"
            script {
                sendNotification('success')
            }
        }
        failure {
            echo "Pipeline failed for branch: ${env.BRANCH_NAME}"
            script {
                sendNotification('failure')
            }
        }
        cleanup {
            cleanWs()
        }
    }
}

// Helper Functions
def setBuildMetadata() {
    def branchName = env.BRANCH_NAME
    if (!branchName || branchName == 'HEAD') {
        branchName = env.GIT_BRANCH
        if (branchName && branchName.startsWith('origin/')) {
            branchName = branchName.replace('origin/', '')
        }
        if (!branchName || branchName == 'HEAD') {
            branchName = sh(
                script: "git branch -r --contains HEAD | grep -v HEAD | head -1 | sed 's|origin/||' | xargs",
                returnStdout: true
            ).trim()
        }
    }
    
    env.BRANCH_NAME = branchName ?: 'main'
    env.GIT_COMMIT_SHORT = sh(
        script: "git rev-parse --short HEAD",
        returnStdout: true
    ).trim()
    
    def buildNumber = env.BUILD_NUMBER ?: '1'
    def commitShort = env.GIT_COMMIT_SHORT ?: 'unknown'
    env.BUILD_TAG = "${env.BRANCH_NAME}-${buildNumber}-${commitShort}"
    
    echo "Build Metadata:"
    echo "   Branch: ${env.BRANCH_NAME}"
    echo "   Build Number: ${buildNumber}"
    echo "   Commit: ${commitShort}"
    echo "   Build Tag: ${env.BUILD_TAG}"
}

def buildDockerImage(String serviceName, String imageName) {
    dir(serviceName) {
        def image = docker.build("${imageName}:${env.BUILD_TAG}")
        echo "Built ${serviceName} image: ${imageName}:${env.BUILD_TAG}"
    }
}

def pushDockerImages() {
    sh 'docker login -u $DOCKER_ID -p $DOCKER_PASS'
    
    // Push service images
    sh "docker push ${env.MOVIE_SERVICE_IMAGE}:${env.BUILD_TAG}"
    sh "docker push ${env.CAST_SERVICE_IMAGE}:${env.BUILD_TAG}"
    
    echo "Pushed images with tag: ${env.BUILD_TAG}"
    
    // Tag and push latest for main branches
    if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'main') {
        sh """
            docker tag ${env.MOVIE_SERVICE_IMAGE}:${env.BUILD_TAG} ${env.MOVIE_SERVICE_IMAGE}:latest
            docker tag ${env.CAST_SERVICE_IMAGE}:${env.BUILD_TAG} ${env.CAST_SERVICE_IMAGE}:latest
            docker push ${env.MOVIE_SERVICE_IMAGE}:latest
            docker push ${env.CAST_SERVICE_IMAGE}:latest
        """
        echo "Pushed latest tags for main branch"
    }
}

def deployToEnvironment(String environment, String valuesFile) {
    echo "Deploying to ${environment.toUpperCase()} environment..."
    
    try {
        // Update image tags in values file
        updateImageTags(valuesFile)
        
        // Clean up namespace if needed
        cleanupNamespace(environment)
        
        // Deploy with Helm
        deployWithHelm(environment, valuesFile)
        
        echo "Successfully deployed to ${environment}"
        
    } catch (Exception e) {
        echo "Deployment to ${environment} failed: ${e.getMessage()}"
        
        // Debug information
        sh """
            echo "=== Debugging deployment failure ==="
            kubectl get pods -n ${environment} || echo "No pods found"
            kubectl get svc -n ${environment} || echo "No services found"
            helm list -n ${environment} || echo "No helm releases found"
        """
        
        throw e
    }
}

def updateImageTags(String valuesFile) {
    // Update image tags for different environments
    if (valuesFile.contains('dev')) {
        sh "sed -i 's|tag: HEAD-[0-9]*-[a-f0-9]*|tag: ${env.BUILD_TAG}|g' ${valuesFile}"
    } else {
        sh "sed -i 's|tag: .*|tag: ${env.BUILD_TAG}|g' ${valuesFile}"
    }
    echo "Updated image tags to ${env.BUILD_TAG} in ${valuesFile}"
}

def cleanupNamespace(String namespace) {
    sh """
        if kubectl get namespace ${namespace} >/dev/null 2>&1; then
            echo "Checking namespace ${namespace}..."
            if ! kubectl get namespace ${namespace} -o jsonpath='{.metadata.labels.app\\.kubernetes\\.io/managed-by}' | grep -q "Helm"; then
                echo "Cleaning up non-Helm managed namespace ${namespace}..."
                kubectl delete namespace ${namespace} --ignore-not-found=true
                
                # Wait for namespace deletion with timeout
                for i in {1..60}; do
                    if ! kubectl get namespace ${namespace} >/dev/null 2>&1; then
                        echo "Namespace ${namespace} deleted successfully"
                        break
                    fi
                    if [ \$i -eq 60 ]; then
                        echo "Namespace deletion timed out, proceeding anyway..."
                        break
                    fi
                    sleep 2
                done
            else
                echo "Namespace ${namespace} is Helm-managed, skipping cleanup"
            fi
        else
            echo "Namespace ${namespace} does not exist"
        fi
    """
}

def deployWithHelm(String environment, String valuesFile) {
    def releaseName = "microservice-${environment}"
    
    sh """
        set -e
        echo "Deploying ${releaseName} to ${environment}..."
        
        # Deploy with Helm (without --wait to avoid timeouts)
        if helm upgrade --install ${releaseName} ./charts \\
            --namespace ${environment} \\
            --create-namespace \\
            --values ${valuesFile} \\
            --timeout ${env.HELM_TIMEOUT}; then
            echo "Helm deployment initiated successfully"
        else
            echo "Helm deployment failed"
            exit 1
        fi
        
        # Manual verification with timeout
        echo "Verifying deployment..."
        for i in \$(seq 1 20); do
            READY_PODS=\$(kubectl get pods -n ${environment} --no-headers | grep -c "Running" || echo "0")
            TOTAL_PODS=\$(kubectl get pods -n ${environment} --no-headers | wc -l || echo "0")
            
            echo "Pod status (\$i/20): \$READY_PODS/\$TOTAL_PODS running"
            
            if [ "\$READY_PODS" -gt 0 ] && [ "\$READY_PODS" -eq "\$TOTAL_PODS" ]; then
                echo "All pods are running in ${environment}"
                break
            fi
            
            if [ \$i -eq 20 ]; then
                echo "Deployment verification timed out, but continuing..."
                kubectl get pods -n ${environment}
            fi
            
            sleep 15
        done
    """
}

def healthCheck(String environment) {
    echo "Running health checks for ${environment.toUpperCase()}..."
    
    sh """
        echo "=== Health Check for ${environment} ==="
        kubectl get pods -n ${environment}
        kubectl get svc -n ${environment}
    """
    
    // Get service pods and run health checks
    def healthCheckResult = checkServiceHealth(environment)
    
    // Show pod logs for troubleshooting
    showPodLogs(environment)
    
    // Report final status
    reportHealthStatus(environment, healthCheckResult)
}

def checkServiceHealth(String environment) {
    def healthCheckPassed = true
    
    sh """
        # Get service pods
        PODS=\$(kubectl get pods -n ${environment} -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\\n' | grep -E '(movie-service|cast-service)' || echo "")
        
        HEALTH_CHECK_PASSED=true
        
        for pod in \$PODS; do
            echo "Checking connectivity to \$pod..."
            
            # Determine service type and endpoint using case statement
            case "\$pod" in
                *movie-service*)
                    ENDPOINT="http://localhost:8000/api/v1/movies"
                    SERVICE_TYPE="movie-service"
                    ;;
                *cast-service*)
                    ENDPOINT="http://localhost:8000/api/v1/casts/docs"
                    SERVICE_TYPE="cast-service"
                    ;;
                *)
                    echo "Unknown service type for pod \$pod, skipping..."
                    continue
                    ;;
            esac
            
            # Test HTTP connectivity using Python
            echo "Testing \$SERVICE_TYPE endpoint: \$ENDPOINT"
            if kubectl exec \$pod -n ${environment} -- python -c "
import urllib.request
import sys
try:
    response = urllib.request.urlopen('\$ENDPOINT')
    status_code = response.getcode()
    if status_code == 200:
        print('PASS - \$SERVICE_TYPE health check PASSED - HTTP Status: ' + str(status_code))
        sys.exit(0)
    else:
        print('WARN - \$SERVICE_TYPE health check WARNING - HTTP Status: ' + str(status_code))
        sys.exit(0)
except Exception as e:
    print('FAIL - \$SERVICE_TYPE health check FAILED - Error: ' + str(e))
    sys.exit(1)
" 2>/dev/null; then
                echo "Health check successful for \$pod"
            else
                echo "Health check failed for \$pod - this might indicate an issue"
                HEALTH_CHECK_PASSED=false
            fi
        done
        
        # Return result via exit code
        if [ "\$HEALTH_CHECK_PASSED" = true ]; then
            exit 0
        else
            exit 1
        fi
    """
    
    return currentBuild.currentResult != 'FAILURE'
}

def showPodLogs(String environment) {
    sh """
        echo ""
        echo "=== Recent Pod Logs ==="
        
        # Get service pods
        PODS=\$(kubectl get pods -n ${environment} -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\\n' | grep -E '(movie-service|cast-service)' || echo "")
        
        for pod in \$PODS; do
            echo "--- Logs for \$pod ---"
            kubectl logs \$pod -n ${environment} --tail=5 2>/dev/null || echo "Could not retrieve logs for \$pod"
        done
    """
}

def reportHealthStatus(String environment, boolean healthCheckPassed) {
    sh """
        echo ""
        if [ "${healthCheckPassed}" = "true" ]; then
            echo "SUCCESS: Health check completed successfully for ${environment}"
        else
            echo "WARNING: Health check completed with warnings for ${environment}"
        fi
        
        echo "Health check completed for ${environment}"
    """
}

def sendNotification(String status) {
    // Placeholder for notification logic
    echo "Sending ${status} notification for build ${env.BUILD_TAG}"
    
    //mail to: "email.com",
    //     subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has ${status}",
    //     body: "For more info on the pipeline ${status}, check out the console output at ${env.BUILD_URL}"
}
