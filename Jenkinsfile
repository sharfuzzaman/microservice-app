pipeline {
    agent any
    environment {
        DOCKER_CONFIG = '/tmp/docker-config'
        DOCKER_HUB_CREDS = credentials('docker-hub-cred')
        GKE_CREDS = credentials('gke-cred')
        PROJECT_ID = 'thesis-work-455913'
        CLUSTER_NAME = 'petclinic-cluster'
        REGION = 'europe-north1'
        CLOUDSDK_PYTHON = '/usr/bin/python3'
        DOCKER_PATH = '/usr/local/bin'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sharfuzzaman/microservice-app.git', branch: 'main'
            }
        }

        stage('Setup GCP Auth') {
            steps {
                script {
                    sh """
                    gcloud auth activate-service-account --key-file=${GKE_CREDS}
                    gcloud config set project ${PROJECT_ID}
                    """
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Secure Docker login with credentials
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                        mkdir -p ${DOCKER_CONFIG}
                        echo '{"auths":{"https://index.docker.io/v1/":{"auth":"$(echo -n ${DOCKER_USER}:${DOCKER_PASS} | base64)"}}}' > ${DOCKER_CONFIG}/config.json
                        ${DOCKER_PATH}/docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} || true
                        """
                    }

                    // Build and push images
                    dir('docker/prometheus') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-prometheus-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-prometheus-server:latest"
                    }

                    dir('docker/grafana') {
                        sh "${DOCKER_PATH}/docker build -t devops8080/spring-petclinic-grafana-server:latest ."
                        sh "${DOCKER_PATH}/docker push devops8080/spring-petclinic-grafana-server:latest"
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                script {
                    // Get cluster credentials with retry logic
                    sh """
                    gcloud container clusters get-credentials ${CLUSTER_NAME} \
                        --region ${REGION} \
                        --project ${PROJECT_ID} || \
                    gcloud container clusters get-credentials ${CLUSTER_NAME} \
                        --zone ${REGION}-a \
                        --project ${PROJECT_ID}
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
                    sh "kubectl get pods --watch"
                }
            }
        }
    }

    post {
        always {
            sh "${DOCKER_PATH}/docker logout || true"
        }
        success {
            echo '✅ Deployment successful! Access your services using:'
            sh """
            echo -n 'API Gateway: ' && kubectl get svc api-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
            echo -n 'Grafana: ' && kubectl get svc grafana-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
            """
        }
        failure {
            echo '❌ Pipeline failed. Check logs above for details.'
            sh "kubectl get events --sort-by='.lastTimestamp'"
        }
    }
}