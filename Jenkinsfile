pipeline {
    agent any

    environment {
        REGISTRY = "localhost:5000"
        NAMESPACE = "devops-project"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/Jagannath-bite/devops-microservices-project.git'
            }
        }

        stage('Build Backend') {
            steps {
                sh 'docker build -t $REGISTRY/backend:latest backend/'
            }
        }

        stage('Build Frontend') {
            steps {
                sh 'docker build -t $REGISTRY/frontend:latest frontend/'
            }
        }

        stage('Push Images') {
            steps {
                sh 'docker push $REGISTRY/backend:latest'
                sh 'docker push $REGISTRY/frontend:latest'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
