pipeline {
    agent any
    environment {
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1-a'
        CLOUDSDK_PYTHON = '/usr/bin/python3' // Force Python 3.9 which is already installed
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
                    // Check if gcloud is already in PATH
                    def gcloudInstalled = sh(script: 'command -v gcloud >/dev/null 2>&1', returnStatus: true) == 0
                    
                    if (!gcloudInstalled) {
                        echo "Installing Google Cloud SDK..."
                        
                        // Download specific version that works with Python 3.9
                        sh '''
                        curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-456.0.0-darwin-arm.tar.gz -o google-cloud-sdk.tar.gz
                        tar -xzf google-cloud-sdk.tar.gz
                        
                        # Install with minimal components
                        ./google-cloud-sdk/install.sh \
                            --quiet \
                            --usage-reporting=false \
                            --path-update=false \
                            --command-completion=false \
                            --override-components=core,gke-gcloud-auth-plugin
                        
                        # Explicitly add to PATH for current session
                        export PATH="$PATH:$PWD/google-cloud-sdk/bin"
                        '''
                    }
                    
                    // Always update PATH in current session
                    sh 'export PATH="$PATH:$PWD/google-cloud-sdk/bin"'
                    
                    // Authenticate with service account
                    sh '''
                    gcloud auth activate-service-account --key-file=$GKE_CREDS
                    gcloud config set project $PROJECT_ID
                    gcloud --version
                    '''
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
                    sh """
                    gcloud container clusters get-credentials $CLUSTER_NAME \
                        --region $REGION \
                        --project $PROJECT_ID
                    """
                    
                    // Apply all Kubernetes manifests
                    def k8sManifests = [
                        'config-server',
                        'discovery-server',
                        'customers-service',
                        'visits-service',
                        'vets-service',
                        'genai-service',
                        'api-gateway',
                        'tracing-server',
                        'admin-server',
                        'grafana-server',
                        'prometheus-server'
                    ]
                    
                    k8sManifests.each { manifest ->
                        sh "kubectl apply -f k8s/${manifest}.yaml"
                    }
                    
                    // Verify deployment
                    sh 'kubectl get pods'
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