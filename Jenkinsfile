pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "vishalinirajika/trend-app"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        AWS_REGION = "ap-south-1"
        EKS_CLUSTER = "trend-eks-cluster"
        REPO_URL = "https://github.com/vishalinirajika-v/Guvi_Trend.git"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: "${REPO_URL}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKERHUB_REPO:$IMAGE_TAG .'
            }
        }

        stage('Test Container Locally') {
            steps {
                sh '''
                    docker rm -f trend-test || true
                    docker run -d --name trend-test -p 3000:80 $DOCKERHUB_REPO:$IMAGE_TAG
                    sleep 10
                    curl -I http://localhost:3000
                    docker rm -f trend-test
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $DOCKERHUB_REPO:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Configure kubeconfig') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    sed -i "s|image: .*|image: $DOCKERHUB_REPO:$IMAGE_TAG|g" k8s/deployment.yaml
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                    kubectl rollout status deployment/trend-deployment
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl get pods
                    kubectl get svc
                    kubectl get deployment
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f trend-test || true'
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}
