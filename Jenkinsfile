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
        KUBECONFIG = "$WORKSPACE/kubeconfig"
        PATH = "/usr/bin:/bin:/usr/sbin:/sbin:${DOCKER_PATH}:$WORKSPACE/google-cloud-sdk/bin"
    }

    stages {
        stage('Install Tools') {
            steps {
                script {
                    sh '''
                    # Install Google Cloud SDK if not available
                    if [ ! -d "google-cloud-sdk" ]; then
                        echo "Installing Google Cloud SDK..."
                        curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-456.0.0-darwin-arm.tar.gz -o google-cloud-sdk.tar.gz
                        tar -xzf google-cloud-sdk.tar.gz
                        rm google-cloud-sdk.tar.gz
                        
                        # Install required components
                        ./google-cloud-sdk/install.sh \\
                            --quiet \\
                            --usage-reporting=false \\
                            --path-update=true \\
                            --command-completion=false
                        
                        ./google-cloud-sdk/bin/gcloud components install \\
                            kubectl \\
                            gke-gcloud-auth-plugin \\
                            beta \\
                            --quiet
                    fi
                    '''
                    
                    sh """
                    # Verify Docker
                    if ! command -v ${DOCKER_PATH}/docker &> /dev/null; then
                        echo "ERROR: Docker not found at ${DOCKER_PATH}/docker"
                        exit 127
                    fi
                    ${DOCKER_PATH}/docker --version
                    
                    # Verify gcloud
                    ./google-cloud-sdk/bin/gcloud --version
                    ./google-cloud-sdk/bin/gke-gcloud-auth-plugin --version
                    """
                }
            }
        }

        stage('Configure GCP Auth') {
            steps {
                script {
                    sh """
                    # Authenticate with service account
                    ./google-cloud-sdk/bin/gcloud auth activate-service-account --key-file=\$GKE_CREDS
                    ./google-cloud-sdk/bin/gcloud config set project \$PROJECT_ID
                    ./google-cloud-sdk/bin/gcloud config set compute/region \$REGION
                    
                    # Get cluster credentials using the auth plugin
                    export USE_GKE_GCLOUD_AUTH_PLUGIN=True
                    ./google-cloud-sdk/bin/gcloud container clusters get-credentials \$CLUSTER_NAME \\
                        --region \$REGION \\
                        --project \$PROJECT_ID
                    
                    # Verify authentication
                    ./google-cloud-sdk/bin/kubectl config current-context
                    ./google-cloud-sdk/bin/kubectl get nodes
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

                    // Prometheus
                    dir('docker/prometheus') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-prometheus-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-prometheus-server:latest"
                    }

                    // Grafana
                    dir('docker/grafana') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-grafana-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-grafana-server:latest"
                    }

                    // Config Server
                    dir('docker/config-server') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-config-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-config-server:latest"
                    }

                    // Discovery Server (Eureka)
                    dir('docker/discovery-server') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-discovery-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-discovery-server:latest"
                    }

                    // API Gateway
                    dir('docker/api-gateway') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-api-gateway:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-api-gateway:latest"
                    }

                    // Customers Service
                    dir('docker/customers-service') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-customers-service:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-customers-service:latest"
                    }

                    // Vets Service
                    dir('docker/vets-service') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-vets-service:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-vets-service:latest"
                    }

                    // Visits Service
                    dir('docker/visits-service') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-visits-service:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-visits-service:latest"
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    sh """
                    # Deploy all components
                    export USE_GKE_GCLOUD_AUTH_PLUGIN=True
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
                    
                    # Verify deployment
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
            sh "./google-cloud-sdk/bin/kubectl get pods || true"
            sh "./google-cloud-sdk/bin/kubectl describe pods || true"
        }
    }
}