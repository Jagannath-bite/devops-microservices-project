pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        CLUSTER_NAME = "devops-cluster"
        ECR_REPO = "robot-shop"
        NAMESPACE = "robot-shop"

        APP_SERVICES = "user cart shipping payment ratings dispatch web"
        INFRA_SERVICES = "mysql mongo redis rabbitmq"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Jagannath-bite/devops-microservices-project.git'
            }
        }

        stage('Verify Repo Structure') {
            steps {
                sh '''
                echo "Repository structure:"
                ls -la
                echo "Services folder:"
                ls -la services || true
                '''
            }
        }

        stage('Configure AWS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                    echo "Account: $ACCOUNT_ID"
                    echo $ACCOUNT_ID > account_id.txt
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    ACCOUNT_ID=$(cat account_id.txt)

                    aws ecr get-login-password --region $AWS_REGION \
                    | docker login \
                    --username AWS \
                    --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Build Application Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    script {

                        def services = env.APP_SERVICES.split(" ")

                        for (svc in services) {

                            sh """
                            ACCOUNT_ID=\$(cat account_id.txt)

                            echo "Building ${svc}"

                            if [ -d services/${svc} ]; then
                                docker build -t ${svc}:${BUILD_NUMBER} services/${svc}
                            elif [ -d ${svc} ]; then
                                docker build -t ${svc}:${BUILD_NUMBER} ${svc}
                            else
                                echo "Skipping ${svc} (folder not found)"
                                exit 0
                            fi

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

        stage('Push Infrastructure Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    script {

                        def infra = env.INFRA_SERVICES.split(" ")

                        for (svc in infra) {

                            sh """
                            ACCOUNT_ID=\$(cat account_id.txt)

                            echo "Processing infra image ${svc}"

                            docker pull ${svc}:latest || docker pull ${svc}:5

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
                    ACCOUNT_ID=$(cat account_id.txt)

                    aws eks update-kubeconfig \
                    --region $AWS_REGION \
                    --name $CLUSTER_NAME

                    kubectl create namespace $NAMESPACE \
                    --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f K8s/ -n $NAMESPACE

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
            echo "Deployment Successful"
        }

        failure {
            echo "Pipeline Failed"
        }
    }
}
