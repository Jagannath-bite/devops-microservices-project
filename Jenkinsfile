pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        ACCOUNT_ID = ""
        ECR_REGISTRY = ""
        NAMESPACE = "robot-shop"
        SERVICES = "user cart shipping payment ratings dispatch web mysql mongo redis rabbitmq"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/Jagannath-bite/devops-microservices-project.git'
            }
        }

        stage('Configure AWS & EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-jenkins-creds'
                ]]) {

                    script {
                        ACCOUNT_ID = sh(
                            script: "aws sts get-caller-identity --query Account --output text",
                            returnStdout: true
                        ).trim()

                        ECR_REGISTRY = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                        sh """
                        aws eks update-kubeconfig --region ${AWS_REGION} --name devops-cluster
                        """

                        echo "Connected to EKS"
                    }
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-jenkins-creds'
                ]]) {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REGISTRY
                    '''
                }
            }
        }

        stage('Build & Push Images') {
            steps {
                script {
                    def services = SERVICES.split(" ")

                    for (svc in services) {

                        echo "Building ${svc}..."

                        if (fileExists("services/${svc}/Dockerfile")) {
                            sh """
                            docker build -t ${svc}:${BUILD_NUMBER} services/${svc}
                            """
                        } else {
                            sh """
                            docker build -t ${svc}:${BUILD_NUMBER} ${svc}
                            """
                        }

                        sh """
                        docker tag ${svc}:${BUILD_NUMBER} \
                        ${ECR_REGISTRY}/robot-shop/${svc}:${BUILD_NUMBER}

                        docker push \
                        ${ECR_REGISTRY}/robot-shop/${svc}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Deploy Base Manifests') {
            steps {
