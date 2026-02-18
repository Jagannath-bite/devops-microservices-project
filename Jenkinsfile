pipeline {
  agent any

  environment {
    REGISTRY = "localhost:5000"
    APP_REPO = "Jagannath-bite/devops-microservices-project"
  }

  stages {

    stage('Checkout') {
      steps {
        git url: "https://github.com/${APP_REPO}.git"
      }
    }

    stage('Build & Push Images') {
      steps {
        script {
          sh 'docker build -t ${REGISTRY}/cart-service:v1 ./services/cart'
          sh 'docker push ${REGISTRY}/cart-service:v1'
        }
      }
    }

    stage('Update K8s Manifests') {
      steps {
        script {
          sh 'sed -i "s|image:.*|image: localhost:5000/cart-service:v1|" ./k8s/cart-deployment.yml'
          sh 'git config user.email "ci@local.dev"'
          sh 'git config user.name "CI"'
          sh 'git add ./k8s/cart-deployment.yml'
          sh 'git commit -m "Update image tag"'
          sh 'git push origin main'
        }
      }
    }

  }
}
