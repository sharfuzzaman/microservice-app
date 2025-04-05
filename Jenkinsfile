pipeline {
    agent any
    environment {
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1-a'
        PATH = "/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:${env.PATH}"
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
                    def pythonInstalled = sh(script: 'python3 --version', returnStatus: true) == 0
                    if (!pythonInstalled) {
                        echo "Python 3 not found. Installing Python 3.12..."
                        sh 'brew install python@3.12'
                        sh 'ln -s /usr/local/opt/python@3.12/bin/python3 /usr/local/bin/python3'
                    } else {
                        echo "Python 3 is already installed."
                        sh 'python3 --version'
                    }
                }
            }
        }
        
        stage('Verify Google Cloud SDK') {
            steps {
                script {
                    // Check if gcloud exists in standard paths or workspace
                    def gcloudInstalled = sh(script: '''
                        if command -v gcloud >/dev/null 2>&1; then
                            exit 0
                        elif [ -f "${WORKSPACE}/google-cloud-sdk/bin/gcloud" ]; then
                            exit 0
                        else
                            exit 1
                        fi
                    ''', returnStatus: true) == 0
                    
                    if (!gcloudInstalled) {
                        echo "Google Cloud SDK not found. Installing..."
                        
                        // Download and install minimal components
                        sh '''
                        curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/google-cloud-sdk.tar.gz -o google-cloud-sdk.tar.gz
                        tar -xzf google-cloud-sdk.tar.gz
                        ./google-cloud-sdk/install.sh \
                            --usage-reporting=false \
                            --path-update=true \
                            --bash-completion=false \
                            --command-completion=false \
                            --quiet \
                            --override-components=core,gke-gcloud-auth-plugin
                        
                        # Add to PATH for current session
                        export PATH="${PATH}:${PWD}/google-cloud-sdk/bin"
                        '''
                    }
                    
                    // Make sure gcloud is in PATH
                    sh 'export PATH="${PATH}:${PWD}/google-cloud-sdk/bin"'
                    
                    // Check if already authenticated
                    def gcloudAuthed = sh(
                        script: 'gcloud auth list --format="value(account)" | grep -q "."',
                        returnStatus: true
                    ) == 0
                    
                    if (!gcloudAuthed) {
                        echo "Authenticating with service account..."
                        sh 'gcloud auth activate-service-account --key-file=$GKE_CREDS'
                    }
                    
                    // Verify installation
                    sh 'gcloud --version'
                    sh 'gcloud config set project $PROJECT_ID'
                }
            }
        }

        stage('Verify Docker') {
            steps {
                script {
                    sh 'docker --version'
                    sh 'command -v docker'
                }
            }
        }
        
        stage('Build and Push Docker Images') {
            steps {
                script {
                    sh 'mkdir -p $DOCKER_CONFIG'
                    sh 'echo \'{"credsStore":""}\' > $DOCKER_CONFIG/config.json'
                    sh 'echo $DOCKER_HUB_CREDS_PSW | docker login -u $DOCKER_HUB_CREDS_USR --password-stdin'
                    
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
                    sh 'kubectl get pods -w'
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
        success {
            echo 'Deployment to GKE completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}