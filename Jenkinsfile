pipeline {
    agent any
    environment {
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1-a'
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sharfuzzaman/microservice-app.git', branch: 'main'
            }
        }
        
        stage('Install Python 3.12') {
            steps {
                script {
                    // Ensure Python 3.12 is installed if it's not already
                    sh '''#!/bin/bash
                    if ! command -v python3 &> /dev/null; then
                        echo "Python 3 is not installed. Installing Python 3.12..."
                        brew install python@3.12
                    else
                        echo "Python 3 is already installed."
                    fi
                    '''
                }
            }
        }
        
        stage('Install Google Cloud SDK') {
            steps {
                script {
                    // Download the Google Cloud SDK tar file
                    sh 'curl -# -f https://dl.google.com/dl/cloudsdk/channels/rapid/google-cloud-sdk.tar.gz -o google-cloud-sdk.tar.gz'
                    
                    // Extract the downloaded tar.gz file
                    sh 'tar -xvzf google-cloud-sdk.tar.gz'
                    
                    // Install Google Cloud SDK and automatically accept the Python 3.12 installation prompt
                    sh '''#!/bin/bash
                    ./google-cloud-sdk/install.sh --usage-reporting=false --path-update=false --bash-completion=false <<EOF
y
EOF
                    '''
                    
                    // Initialize Google Cloud SDK
                    sh './google-cloud-sdk/bin/gcloud init --skip-diagnostics'
                    
                    // Update PATH for the gcloud command
                    sh 'echo "source $(pwd)/google-cloud-sdk/path.bash.inc" >> ~/.bashrc'
                    
                    // Reload the shell configuration
                    sh 'source ~/.bashrc'
                    
                    // Verify Google Cloud SDK installation
                    sh 'gcloud --version'
                }
            }
        }

        stage('Verify Docker') {
            steps {
                script {
                    sh 'echo $PATH'
                    sh '/usr/local/bin/docker --version'
                    sh 'command -v /usr/local/bin/docker'
                }
            }
        }
        
        stage('Build and Push Docker Images') {
            steps {
                script {
                    sh 'mkdir -p /tmp/docker-config && echo \'{"credsStore":""}\' > /tmp/docker-config/config.json'
                    sh 'echo $DOCKER_HUB_CREDS_PSW | /usr/local/bin/docker login -u $DOCKER_HUB_CREDS_USR --password-stdin'
                    dir('docker/prometheus') {
                        sh '/usr/local/bin/docker build -t devops8080/spring-petclinic-prometheus-server:latest .'
                        sh '/usr/local/bin/docker push devops8080/spring-petclinic-prometheus-server:latest'
                    }
                    dir('docker/grafana') {
                        sh '/usr/local/bin/docker build -t devops8080/spring-petclinic-grafana-server:latest .'
                        sh '/usr/local/bin/docker push devops8080/spring-petclinic-grafana-server:latest'
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    sh 'gcloud auth activate-service-account --key-file=$GKE_CREDS'
                    sh "gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION --project $PROJECT_ID"
                    sh 'kubectl apply -f k8s/config-server.yaml'
                    sh 'kubectl apply -f k8s/discovery-server.yaml'
                    sh 'kubectl apply -f k8s/customers-service.yaml'
                    sh 'kubectl apply -f k8s/visits-service.yaml'
                    sh 'kubectl apply -f k8s/vets-service.yaml'
                    sh 'kubectl apply -f k8s/genai-service.yaml'
                    sh 'kubectl apply -f k8s/api-gateway.yaml'
                    sh 'kubectl apply -f k8s/tracing-server.yaml'
                    sh 'kubectl apply -f k8s/admin-server.yaml'
                    sh 'kubectl apply -f k8s/grafana-server.yaml'
                    sh 'kubectl apply -f k8s/prometheus-server.yaml'
                }
            }
        }
    }

    post {
        always {
            script {
                withEnv(["PATH+DOCKER=/usr/local/bin"]) {
                    sh 'docker logout'
                }
            }
        }
        success {
            echo 'Deployment to GKE completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
