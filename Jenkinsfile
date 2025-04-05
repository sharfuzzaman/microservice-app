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
        
        stage('Verify Python 3.12') {
            steps {
                script {
                    def pythonInstalled = sh(script: 'command -v python3.12 || command -v python3', returnStatus: true) == 0
                    if (!pythonInstalled) {
                        echo "Python 3.12 not found. Installing..."
                        sh 'brew install python@3.12'
                    } else {
                        echo "Python 3.12 is already installed."
                        sh 'python3 --version'
                    }
                }
            }
        }
        
        stage('Verify Google Cloud SDK') {
            steps {
                script {
                    def gcloudInstalled = sh(script: 'command -v gcloud', returnStatus: true) == 0
                    if (!gcloudInstalled) {
                        echo "Google Cloud SDK not found. Installing..."
                        // Download the Google Cloud SDK tar file
                        sh 'curl -# -f https://dl.google.com/dl/cloudsdk/channels/rapid/google-cloud-sdk.tar.gz -o google-cloud-sdk.tar.gz'
                        
                        // Extract the downloaded tar.gz file
                        sh 'tar -xvzf google-cloud-sdk.tar.gz'
                        
                        // Install Google Cloud SDK
                        sh '''#!/bin/bash
                        ./google-cloud-sdk/install.sh --usage-reporting=false --path-update=false --bash-completion=false <<EOF
y
EOF
                        '''
                        
                        // Update PATH for the current session
                        sh 'export PATH=$PATH:$(pwd)/google-cloud-sdk/bin'
                    } else {
                        echo "Google Cloud SDK is already installed."
                        sh 'gcloud --version'
                    }
                    
                    // Initialize Google Cloud SDK if not already initialized
                    def gcloudInitialized = sh(script: 'gcloud config list --format="value(core.account)"', returnStatus: true) == 0
                    if (!gcloudInitialized) {
                        echo "Initializing Google Cloud SDK..."
                        sh './google-cloud-sdk/bin/gcloud init --skip-diagnostics'
                    }
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
                    ['config-server', 'discovery-server', 'customers-service', 'visits-service', 
                     'vets-service', 'genai-service', 'api-gateway', 'tracing-server', 
                     'admin-server', 'grafana-server', 'prometheus-server'].each { service ->
                        sh "kubectl apply -f k8s/${service}.yaml"
                    }
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