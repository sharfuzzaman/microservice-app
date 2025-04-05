pipeline {
    agent any
    
    tools {
        gcloud 'gcloud-sdk' // Ensure Google Cloud SDK is set in Global Tool Configuration
        python 'Python 3.12' // Ensure Python 3.12 is set in Global Tool Configuration
    }

    environment {
        // Update the PATH to prioritize Python 3.12 and Docker
        PATH = "/opt/homebrew/opt/python@3.12/libexec/bin:/opt/homebrew/bin:/usr/local/bin:${env.PATH}" // Add Python 3.12 to PATH
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

        stage('Verify Docker') {
            steps {
                script {
                    // Verify Docker installation
                    sh 'echo $PATH'
                    sh '/usr/local/bin/docker --version'
                    sh 'command -v /usr/local/bin/docker'
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Configure Docker login and build images
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
                    // Verify Python version
                    sh '''#!/bin/bash
                    echo "Using Python from: $(which python3)"
                    python3 --version
                    '''
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    try {
                        // Authenticate with Google Cloud and deploy to GKE
                        sh 'gcloud auth activate-service-account --key-file=$GKE_CREDS'
                        sh "gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION --project $PROJECT_ID"
                        
                        // Apply Kubernetes manifests
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
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Docker logout
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
