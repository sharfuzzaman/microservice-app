pipeline {
    agent any
    environment {
        // Set paths for Python, Docker, and Google Cloud SDK tools
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/local/bin/google-cloud-sdk:${env.PATH}" // Ensure the gcloud SDK path is included
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1-a'
        CLOUDSDK_PYTHON = '/opt/homebrew/bin/python3' // Ensure Google Cloud SDK uses Python 3.12
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sharfuzzaman/microservice-app.git', branch: 'main'
            }
        }
        
        stage('Install Google Cloud SDK') {
            steps {
                script {
                    // Check if gcloud is installed, install if not
                    sh '''
                    if ! command -v gcloud &> /dev/null; then
                        echo "gcloud not found! Installing Google Cloud SDK..."
                        curl https://sdk.cloud.google.com | bash
                        source ~/.bash_profile
                        echo "Google Cloud SDK installed."
                    else
                        echo "gcloud is already installed."
                    fi
                    '''
                }
            }
        }

        stage('Install Python 3.12') {
            steps {
                script {
                    // Check if Python 3.12 is installed, if not, install it via Homebrew
                    sh '''
                    if ! command -v python3 &> /dev/null; then
                        echo "Python 3 not found, installing Python 3.12..."
                        brew install python@3.12
                    else
                        echo "Python 3.12 already installed"
                    fi
                    '''
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

        stage('Verify Python and Google Cloud SDK') {
            steps {
                script {
                    // Skip Python prompt if it's already installed
                    sh '''#!/bin/bash
                    if command -v python3 &> /dev/null; then
                        echo "Python 3 is installed. Proceeding with gcloud SDK setup."
                    else
                        echo "Python 3 not found! Please install Python 3.12."
                    fi
                    '''
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    // Authenticate with GKE
                    sh 'gcloud auth activate-service-account --key-file=$GKE_CREDS'
                    sh "gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION --project $PROJECT_ID"
                    
                    // Deploy to GKE
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
                // Logout from Docker
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
