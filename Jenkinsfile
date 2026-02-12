pipeline {
  agent any

  environment {
    AWS_REGION = "us-east-1"
    ECR_REPO = "123456789012.dkr.ecr.us-east-1.amazonaws.com/cart-service"
    IMAGE_TAG = "v${BUILD_NUMBER}"
  }

  stages {

    stage('Clone') {
      steps {
        git 'https://github.com/Jagannath-bite/devops-microservices-project.git'
      }
    }

    stage('Build Image') {
      steps {
        dir('services/cart') {
          sh 'docker build -t cart-service:$IMAGE_TAG .'
        }
      }
    }

    stage('Login to ECR') {
      steps {
        sh '''
        aws ecr get-login-password --region $AWS_REGION | \
        docker login --username AWS --password-stdin $ECR_REPO
        '''
      }
    }

    stage('Tag & Push') {
      steps {
        sh '''
        docker tag cart-service:$IMAGE_TAG $ECR_REPO:$IMAGE_TAG
        docker push $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
        sed -i "s|IMAGE_TAG|$IMAGE_TAG|g" k8s/cart/deployment.yaml
        kubectl apply -f k8s/cart/
        '''
      }
    }
  }
}
