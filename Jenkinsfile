pipeline {
    agent any
    tools {
        gcloud 'gcloud-sdk'
    }
    environment {
        PATH = "/opt/homebrew/bin:${env.PATH}"
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
        stage ('Verify gcloud'){
            steps {
                script {
                    // Example of verifying the path and checking if gcloud works
                    sh '''#!/bin/bash
                    echo "Current PATH: $PATH"
                    gcloud --version
                    '''
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