pipeline {
    agent any
    environment {
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1'
        CLOUDSDK_PYTHON = '/usr/bin/python3'
        DOCKER_PATH = '/usr/local/bin'
        GOOGLE_APPLICATION_CREDENTIALS = credentials('gke-cred') // For direct auth
    }

    stages {
        stage('Install Tools') {
            steps {
                script {
                    // Install Google Cloud SDK with all required components
                    sh '''
                    if ! command -v gcloud &> /dev/null; then
                        echo "Installing Google Cloud SDK..."
                        curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-456.0.0-darwin-arm.tar.gz -o google-cloud-sdk.tar.gz
                        tar -xzf google-cloud-sdk.tar.gz
                        ./google-cloud-sdk/install.sh \
                            --quiet \
                            --usage-reporting=false \
                            --path-update=true \
                            --command-completion=false
                        
                        # Install required components
                        ./google-cloud-sdk/bin/gcloud components install \
                            kubectl \
                            gke-gcloud-auth-plugin \
                            --quiet
                        
                        # Update PATH for current session
                        export PATH="$PATH:$PWD/google-cloud-sdk/bin"
                        source ~/.bash_profile
                    fi
                    '''
                    
                    // Verify Docker
                    sh """
                    if ! command -v ${DOCKER_PATH}/docker &> /dev/null; then
                        echo "ERROR: Docker not found at ${DOCKER_PATH}/docker"
                        exit 127
                    fi
                    """
                }
            }
        }

        stage('Configure GCP Auth') {
            steps {
                script {
                    // Authenticate using service account
                    sh """
                    ./google-cloud-sdk/bin/gcloud auth activate-service-account --key-file=\$GKE_CREDS
                    ./google-cloud-sdk/bin/gcloud config set project \$PROJECT_ID
                    ./google-cloud-sdk/bin/gcloud config set compute/region \$REGION
                    
                    # Configure kubectl to use direct authentication
                    ./google-cloud-sdk/bin/gcloud container clusters get-credentials \$CLUSTER_NAME \\
                        --region \$REGION \\
                        --project \$PROJECT_ID \\
                        --internal-ip
                    """
                }
            }
        }

        stage('Build and Push Images') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                        mkdir -p \$DOCKER_CONFIG
                        echo '{"auths":{"https://index.docker.io/v1/":{"auth":"\$(echo -n \$DOCKER_USER:\$DOCKER_PASS | base64)"}}}' > \$DOCKER_CONFIG/config.json
                        ${DOCKER_PATH}/docker login -u \$DOCKER_USER -p \$DOCKER_PASS
                        """
                    }

                    dir('docker/prometheus') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-prometheus-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-prometheus-server:latest"
                    }

                    dir('docker/grafana') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-grafana-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-grafana-server:latest"
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    // Use direct authentication with service account
                    sh """
                    export GOOGLE_APPLICATION_CREDENTIALS=\$GKE_CREDS
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/config-server.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/discovery-server.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/customers-service.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/visits-service.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/vets-service.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/genai-service.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/api-gateway.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/tracing-server.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/admin-server.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/grafana-server.yaml
                    ./google-cloud-sdk/bin/kubectl apply -f k8s/prometheus-server.yaml
                    
                    ./google-cloud-sdk/bin/kubectl get pods
                    """
                }
            }
        }
    }

    post {
        always {
            sh "${DOCKER_PATH}/docker logout || true"
        }
        success {
            echo '✅ Deployment successful!'
            sh """
            echo 'API Gateway: ' \$(./google-cloud-sdk/bin/kubectl get svc api-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
            echo 'Grafana: ' \$(./google-cloud-sdk/bin/kubectl get svc grafana-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
            """
        }
        failure {
            echo '❌ Pipeline failed. Check logs for details.'
            sh "./google-cloud-sdk/bin/kubectl get events --sort-by=.lastTimestamp || true"
        }
    }
}