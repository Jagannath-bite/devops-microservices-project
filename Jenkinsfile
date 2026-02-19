pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        NAMESPACE = "robot-shop"
        SERVICES = "user cart shipping payment ratings dispatch web mysql mongo redis rabbitmq"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Jagannath-bite/devops-microservices-project.git'
            }
        }

        stage('Configure AWS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                    echo "Account ID: $ACCOUNT_ID"
                    echo "ACCOUNT_ID=$ACCOUNT_ID" > account.env
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    source account.env
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Build & Push Images') {
            steps {
                script {
                    def services = env.SERVICES.split(" ")

                    for (svc in services) {

                        sh """
                        ACCOUNT_ID=\$(cat account.env | cut -d '=' -f2)

                        if [ -d services/${svc} ]; then
                            docker build -t ${svc}:${BUILD_NUMBER} services/${svc}
                        else
                            docker build -t ${svc}:${BUILD_NUMBER} ${svc}
                        fi

                        docker tag ${svc}:${BUILD_NUMBER} \
                        \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/robot-shop/${svc}:${BUILD_NUMBER}

                        docker push \
                        \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/robot-shop/${svc}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    ACCOUNT_ID=$(cat account.env | cut -d '=' -f2)

                    aws eks update-kubeconfig --region $AWS_REGION --name devops-cluster

                    kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f k8/ -n $NAMESPACE

                    for svc in $SERVICES
                    do
                      kubectl set image deployment/$svc \
                      $svc=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/robot-shop/$svc:$BUILD_NUMBER \
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
