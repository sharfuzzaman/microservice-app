pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDS = credentials('docker-hub-credentials')
        GKE_CREDS = credentials('gke-credentials')
        PROJECT_ID = 'your-project-id'  // Replace with your GCP project ID
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'us-central1'
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/yourusername/your-repo.git', branch: 'main'
            }
        }
        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Log in to Docker Hub
                    sh 'echo $DOCKER_HUB_CREDS_PSW | docker login -u $DOCKER_HUB_CREDS_USR --password-stdin'
                    
                    // Build and push prometheus-server
                    dir('docker/prometheus') {
                        sh 'docker build -t sharfuzzamansajib/spring-petclinic-prometheus-server:latest .'
                        sh 'docker push sharfuzzamansajib/spring-petclinic-prometheus-server:latest'
                    }
                    
                    // Build and push grafana-server
                    dir('docker/grafana') {
                        sh 'docker build -t sharfuzzamansajib/spring-petclinic-grafana-server:latest .'
                        sh 'docker push sharfuzzamansajib/spring-petclinic-grafana-server:latest'
                    }
                }
            }
        }
        stage('Deploy to GKE') {
            steps {
                script {
                    // Authenticate with GKE
                    sh 'gcloud auth activate-service-account --key-file=$GKE_CREDS'
                    sh "gcloud container clusters get-credentials $CLUSTER_NAME --region $REGION --project $PROJECT_ID"
                    
                    // Apply Kubernetes manifests
                    sh 'kubectl apply -f config-server.yaml'
                    sh 'kubectl apply -f discovery-server.yaml'
                    sh 'kubectl apply -f customers-service.yaml'
                    sh 'kubectl apply -f visits-service.yaml'
                    sh 'kubectl apply -f vets-service.yaml'
                    sh 'kubectl apply -f genai-service.yaml'
                    sh 'kubectl apply -f api-gateway.yaml'
                    sh 'kubectl apply -f tracing-server.yaml'
                    sh 'kubectl apply -f admin-server.yaml'
                    sh 'kubectl apply -f grafana-server.yaml'
                    sh 'kubectl apply -f prometheus-server.yaml'
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