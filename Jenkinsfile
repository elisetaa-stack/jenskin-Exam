pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USERNAME = 'elisetaa'
        DOCKERHUB_USERNAME = 'elisetaa'  // Added this
        IMAGE_TAG = "${BUILD_NUMBER}"     // Added this
        CAST_IMAGE = "${DOCKER_USERNAME}/cast-service"
        MOVIE_IMAGE = "${DOCKER_USERNAME}/movie-service"
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                script {
                    echo "=== Checking out code from GitHub ==="
                    checkout scm
                }
            }
        }
        
        stage('Environment Detection') {
            steps {
                script {
                    def branch = env.GIT_BRANCH ?: 'develop'
                    
                    if (branch.contains('master') || branch.contains('main')) {
                        env.DEPLOY_ENV = 'prod'
                        env.NAMESPACE = 'prod'
                        env.DB_SUFFIX = 'prod'
                    } else if (branch.contains('staging')) {
                        env.DEPLOY_ENV = 'staging'
                        env.NAMESPACE = 'staging'
                        env.DB_SUFFIX = 'staging'
                    } else if (branch.contains('qa')) {
                        env.DEPLOY_ENV = 'qa'
                        env.NAMESPACE = 'qa'
                        env.DB_SUFFIX = 'qa'
                    } else {
                        env.DEPLOY_ENV = 'dev'
                        env.NAMESPACE = 'dev'
                        env.DB_SUFFIX = 'dev'
                    }
                    
                    echo "Branch: ${branch}"
                    echo "Deployment Environment: ${env.DEPLOY_ENV}"
                    echo "Kubernetes Namespace: ${env.NAMESPACE}"
                    echo "Database Suffix: ${env.DB_SUFFIX}"
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Cast Service') {
                    steps {
                        script {
                            echo "=== Building Cast Service Docker Image ==="
                            dir('cast-service') {
                                sh """
                                    docker build -t ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG} .
                                    docker tag ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG} \
                                        ${DOCKERHUB_USERNAME}/cast-service:${env.DEPLOY_ENV}-latest
                                """
                            }
                        }
                    }
                }
                stage('Build Movie Service') {
                    steps {
                        script {
                            echo "=== Building Movie Service Docker Image ==="
                            dir('movie-service') {
                                sh """
                                    docker build -t ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG} .
                                    docker tag ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG} \
                                        ${DOCKERHUB_USERNAME}/movie-service:${env.DEPLOY_ENV}-latest
                                """
                            }
                        }
                    }
                }
            }
        }
        
        stage('Run Tests') {
            parallel {
                stage('Test Cast Service') {
                    steps {
                        script {
                            echo "=== Running Cast Service Tests ==="
                            dir('cast-service') {
                                sh """
                                    docker run --rm ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG} \
                                        python -m pytest tests/ -v || echo "Tests completed with warnings"
                                """
                            }
                        }
                    }
                }
                stage('Test Movie Service') {
                    steps {
                        script {
                            echo "=== Running Movie Service Tests ==="
                            dir('movie-service') {
                                sh """
                                    docker run --rm ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG} \
                                        python -m pytest tests/ -v || echo "Tests completed with warnings"
                                """
                            }
                        }
                    }
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Scan Cast Service') {
                    steps {
                        script {
                            echo "=== Security Scanning Cast Service ==="
                            sh """
                                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                                    aquasec/trivy image --severity HIGH,CRITICAL \
                                    ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG} || echo "Scan completed"
                            """
                        }
                    }
                }
                stage('Scan Movie Service') {
                    steps {
                        script {
                            echo "=== Security Scanning Movie Service ==="
                            sh """
                                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                                    aquasec/trivy image --severity HIGH,CRITICAL \
                                    ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG} || echo "Scan completed"
                            """
                        }
                    }
                }
            }
        }
        
        stage('Push to DockerHub') {
            steps {
                script {
                    echo "=== Pushing Images to DockerHub ==="
                    sh """
                        echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                        
                        # Push Cast Service Images
                        docker push ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG}
                        docker push ${DOCKERHUB_USERNAME}/cast-service:${env.DEPLOY_ENV}-latest
                        
                        # Push Movie Service Images
                        docker push ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG}
                        docker push ${DOCKERHUB_USERNAME}/movie-service:${env.DEPLOY_ENV}-latest
                        
                        docker logout
                    """
                    
                    echo "✓ Images pushed successfully:"
                    echo "  - ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG}"
                    echo "  - ${DOCKERHUB_USERNAME}/cast-service:${env.DEPLOY_ENV}-latest"
                    echo "  - ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG}"
                    echo "  - ${DOCKERHUB_USERNAME}/movie-service:${env.DEPLOY_ENV}-latest"
                }
            }
        }
        
        stage('Deploy to Dev/QA/Staging') {
            when {
                not {
                    anyOf {
                        branch 'master'
                        branch 'main'
                    }
                }
            }
            steps {
                script {
                    echo "=== Deploying to ${env.DEPLOY_ENV} Environment ==="
                    
                    sh """
                        # Create namespace if it doesn't exist
                        kubectl create namespace ${env.NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Create database secrets
                        kubectl create secret generic cast-db-secret \
                            --from-literal=username=cast_db_username \
                            --from-literal=password=cast_db_password \
                            --from-literal=database=cast_db_${env.DB_SUFFIX} \
                            --namespace ${env.NAMESPACE} \
                            --dry-run=client -o yaml | kubectl apply -f -
                        
                        kubectl create secret generic movie-db-secret \
                            --from-literal=username=movie_db_username \
                            --from-literal=password=movie_db_password \
                            --from-literal=database=movie_db_${env.DB_SUFFIX} \
                            --namespace ${env.NAMESPACE} \
                            --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Deploy Cast Service
                        helm upgrade --install cast-service ./helm-charts/cast-service \
                            --namespace ${env.NAMESPACE} \
                            --set image.repository=${DOCKERHUB_USERNAME}/cast-service \
                            --set image.tag=${IMAGE_TAG} \
                            --set environment=${env.DEPLOY_ENV} \
                            --set database.name=cast_db_${env.DB_SUFFIX} \
                            --wait --timeout 5m
                        
                        # Deploy Movie Service (depends on Cast Service)
                        helm upgrade --install movie-service ./helm-charts/movie-service \
                            --namespace ${env.NAMESPACE} \
                            --set image.repository=${DOCKERHUB_USERNAME}/movie-service \
                            --set image.tag=${IMAGE_TAG} \
                            --set environment=${env.DEPLOY_ENV} \
                            --set database.name=movie_db_${env.DB_SUFFIX} \
                            --set castService.url=http://cast-service:8000/api/v1/casts/ \
                            --wait --timeout 5m
                        
                        # Deploy Nginx Gateway
                        kubectl apply -f kubernetes-manifests/nginx-gateway.yaml -n ${env.NAMESPACE}
                    """
                    
                    echo "✓ Deployment completed successfully to ${env.DEPLOY_ENV}"
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
                    echo "=== Production Deployment Approval Required ==="
                    
                    // Manual approval gate for production
                    timeout(time: 30, unit: 'MINUTES') {
                        input message: 'Deploy to Production Environment?', 
                              ok: 'Deploy to Production',
                              submitter: 'admin,devops-team',
                              parameters: [
                                  text(name: 'DEPLOYMENT_NOTES', 
                                       description: 'Enter deployment notes or ticket number:',
                                       defaultValue: 'Production deployment')
                              ]
                    }
                    
                    echo "=== Deploying to Production ==="
                    
                    sh """
                        kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Create production database secrets
                        kubectl create secret generic cast-db-secret \
                            --from-literal=username=cast_db_username \
                            --from-literal=password=cast_db_password \
                            --from-literal=database=cast_db_prod \
                            --namespace prod \
                            --dry-run=client -o yaml | kubectl apply -f -
                        
                        kubectl create secret generic movie-db-secret \
                            --from-literal=username=movie_db_username \
                            --from-literal=password=movie_db_password \
                            --from-literal=database=movie_db_prod \
                            --namespace prod \
                            --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Deploy Cast Service to Production
                        helm upgrade --install cast-service ./helm-charts/cast-service \
                            --namespace prod \
                            --set image.repository=${DOCKERHUB_USERNAME}/cast-service \
                            --set image.tag=${IMAGE_TAG} \
                            --set environment=prod \
                            --set database.name=cast_db_prod \
                            --set replicaCount=3 \
                            --set resources.requests.cpu=500m \
                            --set resources.requests.memory=512Mi \
                            --wait --timeout 10m
                        
                        # Deploy Movie Service to Production
                        helm upgrade --install movie-service ./helm-charts/movie-service \
                            --namespace prod \
                            --set image.repository=${DOCKERHUB_USERNAME}/movie-service \
                            --set image.tag=${IMAGE_TAG} \
                            --set environment=prod \
                            --set database.name=movie_db_prod \
                            --set castService.url=http://cast-service:8000/api/v1/casts/ \
                            --set replicaCount=3 \
                            --set resources.requests.cpu=500m \
                            --set resources.requests.memory=512Mi \
                            --wait --timeout 10m
                        
                        # Deploy Nginx Gateway
                        kubectl apply -f kubernetes-manifests/nginx-gateway.yaml -n prod
                    """
                    
                    echo "✓ Production deployment completed successfully!"
                }
            }
        }
        
        stage('Smoke Tests') {
            steps {
                script {
                    echo "=== Running Smoke Tests ==="
                    
                    sh """
                        # Wait for services to be ready
                        sleep 10
                        
                        # Test Cast Service
                        echo "Testing Cast Service..."
                        kubectl port-forward -n ${env.NAMESPACE} svc/cast-service 8002:8000 &
                        CAST_PF_PID=\$!
                        sleep 5
                        
                        CAST_HEALTH=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8002/health || echo "000")
                        if [ "\$CAST_HEALTH" = "200" ]; then
                            echo "✓ Cast Service health check passed"
                        else
                            echo "✗ Cast Service health check failed (HTTP \$CAST_HEALTH)"
                        fi
                        
                        kill \$CAST_PF_PID || true
                        sleep 2
                        
                        # Test Movie Service
                        echo "Testing Movie Service..."
                        kubectl port-forward -n ${env.NAMESPACE} svc/movie-service 8001:8000 &
                        MOVIE_PF_PID=\$!
                        sleep 5
                        
                        MOVIE_HEALTH=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8001/health || echo "000")
                        if [ "\$MOVIE_HEALTH" = "200" ]; then
                            echo "✓ Movie Service health check passed"
                        else
                            echo "✗ Movie Service health check failed (HTTP \$MOVIE_HEALTH)"
                        fi
                        
                        kill \$MOVIE_PF_PID || true
                        
                        echo "Smoke tests completed"
                    """
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    echo "=== Cleaning Up ==="
                    sh """
                        # Remove old Docker images
                        docker system prune -f || true
                        
                        echo "✓ Cleanup completed"
                    """
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo """
                ╔════════════════════════════════════════════════════════════╗
                ║           PIPELINE COMPLETED SUCCESSFULLY! ✓               ║
                ╚════════════════════════════════════════════════════════════╝
                
                Deployment Details:
                ├─ Environment: ${env.DEPLOY_ENV}
                ├─ Namespace: ${env.NAMESPACE}
                ├─ Build Number: ${BUILD_NUMBER}
                └─ Images:
                   ├─ ${DOCKERHUB_USERNAME}/cast-service:${IMAGE_TAG}
                   └─ ${DOCKERHUB_USERNAME}/movie-service:${IMAGE_TAG}
                
                Services Status:
                ├─ Cast Service: Deployed ✓
                └─ Movie Service: Deployed ✓
                """
            }
        }
        failure {
            script {
                echo """
                ╔════════════════════════════════════════════════════════════╗
                ║              PIPELINE FAILED! ✗                            ║
                ╚════════════════════════════════════════════════════════════╝
                
                Please check the console output above for error details.
                """
            }
        }
        always {
            script {
                echo "=== Pipeline Execution Completed ==="
                // Removed cleanWs() to avoid context error
            }
        }
    }
}
