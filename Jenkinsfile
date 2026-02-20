pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        CLUSTER_NAME = "devops-cluster"
        NAMESPACE = "robot-shop"
        REPO_NAME = "robot-shop"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Jagannath-bite/devops-microservices-project.git'
            }
        }

        stage('Detect Services') {
            steps {
                script {
                    APP_SERVICES = sh(
                        script: "ls services",
                        returnStdout: true
                    ).trim().split()
                }
            }
        }

        stage('Configure AWS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    aws sts get-caller-identity
                    '''
                }
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

        stage('Create ECR Repositories') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    script {
                        for (svc in APP_SERVICES) {
                            sh """
                            ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)

                            aws ecr describe-repositories \
                              --repository-names ${REPO_NAME}/${svc} \
                              --region ${AWS_REGION} || \

                            aws ecr create-repository \
                              --repository-name ${REPO_NAME}/${svc} \
                              --region ${AWS_REGION}
                            """
                        }
                    }
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    script {
                        for (svc in APP_SERVICES) {

                            sh """
                            ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)

                            echo "Building ${svc}"

                            docker build -t ${svc}:${BUILD_NUMBER} services/${svc}

                            docker tag ${svc}:${BUILD_NUMBER} \
                            \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}/${svc}:${BUILD_NUMBER}

                            docker push \
                            \$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}/${svc}:${BUILD_NUMBER}
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

                    aws eks update-kubeconfig \
                    --region $AWS_REGION \
                    --name $CLUSTER_NAME

                    kubectl create namespace $NAMESPACE \
                    --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f K8s/ -n $NAMESPACE

                    for svc in $(ls services)
                    do
                      kubectl set image deployment/$svc \
                      $svc=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO_NAME/$svc:$BUILD_NUMBER \
                      -n $NAMESPACE || true
                    done
                    '''
                }
            }
        }

    }

    post {

        success {
            echo "Deployment Successful"
        }

        failure {
            echo "Pipeline Failed"
        }
    }
}
