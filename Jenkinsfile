pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        NAMESPACE = "robot-shop"
        APP_SERVICES = "user cart shipping payment ratings dispatch web"
        INFRA_SERVICES = "mysql mongo redis rabbitmq"
        CLUSTER_NAME = "devops-cluster"
        ECR_REPO = "robot-shop"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Jagannath-bite/devops-microservices-project.git'
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin \
                    $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Build & Push Application Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    script {
                        def services = env.APP_SERVICES.split(" ")

                        for (svc in services) {
                            sh """
                            ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)

                            echo "Building service: ${svc}"

                            docker build -t ${svc}:${BUILD_NUMBER} services/${svc}

                            docker tag ${svc}:${BUILD_NUMBER} \
                            \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}/${svc}:${BUILD_NUMBER}

                            docker push \
                            \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}/${svc}:${BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }

        stage('Push Infrastructure Images to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    script {
                        def infra = env.INFRA_SERVICES.split(" ")

                        for (svc in infra) {
                            sh """
                            ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)

                            echo "Processing infra service: ${svc}"

                            docker pull ${svc}:latest || docker pull ${svc}:5 || true

                            docker tag ${svc}:latest \
                            \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}/${svc}:latest || true

                            docker push \
                            \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}/${svc}:latest || true
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                    aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

                    kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f k8/ -n $NAMESPACE

                    for svc in $APP_SERVICES
                    do
                      kubectl set image deployment/$svc \
                      $svc=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO/$svc:$BUILD_NUMBER \
                      -n $NAMESPACE || true
                    done
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful 🚀'
        }
        failure {
            echo 'Pipeline Failed ❌'
        }
    }
}
