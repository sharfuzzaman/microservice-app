pipeline {
    agent any  // This assigns an agent to the entire pipeline
    environment {
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-credentials')
        PROJECT_ID = 'thesis-work-455913'  // Replace with your GCP project ID
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1-a'
    }
    stages {
        stage('Checkout') {
            steps {
                // Checkout is fine without explicit node since agent any is at pipeline level
                git url: 'https://github.com/yourusername/spring-petclinic-microservices.git', branch: 'main'
            }
        }
        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Explicitly wrap sh steps in a node block if needed
                    sh 'echo $DOCKER_HUB_CREDS_PSW | docker login -u $DOCKER_HUB_CREDS_USR --password-stdin'
                    dir('docker/prometheus') {
                        sh 'docker build -t sharfuzzamansajib/spring-petclinic-prometheus-server:latest .'
                        sh 'docker push sharfuzzamansajib/spring-petclinic-prometheus-server:latest'
                    }
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