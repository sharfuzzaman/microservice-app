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
                        ./google-cloud-sdk/install.sh --quiet \
                            --usage-reporting=false \
                            --path-update=true
                        source ~/.bashrc
                    fi
                    '''
                    
                    // Verify installations
                    sh '''
                    gcloud --version || exit 1
                    kubectl version --client || exit 1
                    docker --version || exit 1
                    '''
                }
            }
        }

        stage('Setup GCP Auth') {
            steps {
                script {
                    sh """
                    gcloud auth activate-service-account --key-file=\$GKE_CREDS
                    gcloud config set project \$PROJECT_ID
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
                        \$DOCKER_PATH/docker login -u \$DOCKER_USER -p \$DOCKER_PASS || true
                        """
                    }

                    dir('docker/prometheus') {
                        sh "\$DOCKER_PATH/docker build -t devops8080/spring-petclinic-prometheus-server:latest ."
                        sh "\$DOCKER_PATH/docker push devops8080/spring-petclinic-prometheus-server:latest"
                    }

                    dir('docker/grafana') {
                        sh "\$DOCKER_PATH/docker build -t devops8080/spring-petclinic-grafana-server:latest ."
                        sh "\$DOCKER_PATH/docker push devops8080/spring-petclinic-grafana-server:latest"
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    sh """
                    gcloud container clusters get-credentials \$CLUSTER_NAME \\
                        --region \$REGION \\
                        --project \$PROJECT_ID || \\
                    gcloud container clusters get-credentials \$CLUSTER_NAME \\
                        --zone \$REGION-a \\
                        --project \$PROJECT_ID
                    """

                    def k8sManifests = [
                        'config-server', 'discovery-server', 'customers-service',
                        'visits-service', 'vets-service', 'genai-service',
                        'api-gateway', 'tracing-server', 'admin-server',
                        'grafana-server', 'prometheus-server'
                    ]

                    k8sManifests.each { manifest ->
                        sh "kubectl apply -f k8s/${manifest}.yaml"
                    }

                    sh "kubectl get pods --watch"
                }
            }
        }
    }

    post {
        always {
            sh "\$DOCKER_PATH/docker logout || true"
        }
        success {
            echo '✅ Deployment successful!'
            sh """
            echo 'API Gateway IP: ' \$(kubectl get svc api-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
            echo 'Grafana IP: ' \$(kubectl get svc grafana-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
            """
        }
        failure {
            echo '❌ Pipeline failed'
            sh "kubectl get events --sort-by=.lastTimestamp || true"
        }
    }
}