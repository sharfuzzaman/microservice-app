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
        DOCKER_PATH = '/usr/local/bin'  // Confirmed from your which docker output
    }

    stages {
        stage('Install Prerequisites') {
            steps {
                script {
                    // Install Google Cloud SDK if not available
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
                        source ~/.bashrc
                    fi
                    '''
                    
                    // Verify Docker is installed and accessible
                    sh """
                    if ! command -v docker &> /dev/null; then
                        echo "ERROR: Docker not found at \${DOCKER_PATH}/docker"
                        echo "Please install Docker Desktop for Mac and ensure it's in your PATH"
                        exit 127
                    fi
                    """
                    
                    // Verify all tools
                    sh """
                    $PWD/google-cloud-sdk/bin/gcloud --version || exit 1
                    $PWD/google-cloud-sdk/bin/kubectl version --client || exit 1
                    ${DOCKER_PATH}/docker --version || exit 1
                    """
                }
            }
        }

        stage('Setup GCP Auth') {
            steps {
                script {
                    sh """
                    $PWD/google-cloud-sdk/bin/gcloud auth activate-service-account --key-file=\$GKE_CREDS
                    $PWD/google-cloud-sdk/bin/gcloud config set project \$PROJECT_ID
                    """
                }
            }
        }

        stage('Build and Push Docker Images') {
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
                        ${DOCKER_PATH}/docker login -u \$DOCKER_USER -p \$DOCKER_PASS || true
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
                    sh """
                    $PWD/google-cloud-sdk/bin/gcloud container clusters get-credentials \$CLUSTER_NAME \\
                        --region \$REGION \\
                        --project \$PROJECT_ID
                    """

                    sh """
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/config-server.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/discovery-server.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/customers-service.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/visits-service.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/vets-service.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/genai-service.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/api-gateway.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/tracing-server.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/admin-server.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/grafana-server.yaml
                    $PWD/google-cloud-sdk/bin/kubectl apply -f k8s/prometheus-server.yaml
                    
                    $PWD/google-cloud-sdk/bin/kubectl get pods
                    """
                }
            }
        }
    }

    post {
        always {
            sh "${DOCKER_PATH}/docker logout || true"
        }
    }
}