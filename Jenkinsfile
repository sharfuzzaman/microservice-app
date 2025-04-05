pipeline {
    agent any
    environment {
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1-a'
        CLOUDSDK_PYTHON = '/usr/bin/python3' // Force use of Python 3.9
    }
    
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sharfuzzaman/microservice-app.git', branch: 'main'
            }
        }
        
        stage('Verify Python') {
            steps {
                script {
                    sh 'python3 --version'
                }
            }
        }
        
        stage('Setup Google Cloud SDK') {
            steps {
                script {
                    // Check if gcloud exists in standard paths
                    def gcloudInstalled = sh(script: 'command -v gcloud >/dev/null 2>&1', returnStatus: true) == 0
                    
                    if (!gcloudInstalled) {
                        echo "Installing Google Cloud SDK..."
                        
                        // Download and install minimal components
                        sh '''
                        # Remove any existing installation
                        rm -rf google-cloud-sdk google-cloud-sdk.tar.gz
                        
                        # Download specific version compatible with Python 3.9
                        curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-456.0.0-darwin-arm.tar.gz -o google-cloud-sdk.tar.gz
                        tar -xzf google-cloud-sdk.tar.gz
                        
                        # Install with no additional components
                        ./google-cloud-sdk/install.sh \
                            --quiet \
                            --usage-reporting=false \
                            --path-update=false \
                            --command-completion=false
                        
                        # Add to PATH for current session
                        export PATH="$PATH:$PWD/google-cloud-sdk/bin"
                        
                        # Install required components separately
                        ./google-cloud-sdk/bin/gcloud components install kubectl gke-gcloud-auth-plugin --quiet
                        '''
                    }
                    
                    // Ensure PATH is set for all commands
                    withEnv(["PATH+CLOUD=${pwd()}/google-cloud-sdk/bin"]) {
                        sh '''
                        gcloud --version
                        gcloud auth activate-service-account --key-file=$GKE_CREDS
                        gcloud config set project $PROJECT_ID
                        '''
                    }
                }
            }
        }

        stage('Verify Docker') {
            steps {
                script {
                    sh 'docker --version'
                }
            }
        }
        
        stage('Build and Push Docker Images') {
            steps {
                script {
                    sh '''
                    mkdir -p $DOCKER_CONFIG
                    echo '{"credsStore":""}' > $DOCKER_CONFIG/config.json
                    echo $DOCKER_HUB_CREDS_PSW | docker login -u $DOCKER_HUB_CREDS_USR --password-stdin
                    '''
                    
                    dir('docker/prometheus') {
                        sh 'docker build -t devops8080/spring-petclinic-prometheus-server:latest .'
                        sh 'docker push devops8080/spring-petclinic-prometheus-server:latest'
                    }
                    
                    dir('docker/grafana') {
                        sh 'docker build -t devops8080/spring-petclinic-grafana-server:latest .'
                        sh 'docker push devops8080/spring-petclinic-grafana-server:latest'
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    withEnv(["PATH+CLOUD=${pwd()}/google-cloud-sdk/bin"]) {
                        sh """
                        gcloud container clusters get-credentials $CLUSTER_NAME \
                            --region $REGION \
                            --project $PROJECT_ID
                        
                        # Apply all Kubernetes manifests
                        kubectl apply -f k8s/config-server.yaml
                        kubectl apply -f k8s/discovery-server.yaml
                        kubectl apply -f k8s/customers-service.yaml
                        kubectl apply -f k8s/visits-service.yaml
                        kubectl apply -f k8s/vets-service.yaml
                        kubectl apply -f k8s/genai-service.yaml
                        kubectl apply -f k8s/api-gateway.yaml
                        kubectl apply -f k8s/tracing-server.yaml
                        kubectl apply -f k8s/admin-server.yaml
                        kubectl apply -f k8s/grafana-server.yaml
                        kubectl apply -f k8s/prometheus-server.yaml
                        
                        # Verify deployment
                        kubectl get pods
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo 'Deployment to GKE completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}