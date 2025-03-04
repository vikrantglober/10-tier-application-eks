pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_ACCOUNT_ID = "084828580507"
        AWS_REGION = "ap-south-1"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vikrantglober/10-tier-application-eks.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=10-Tier \
                        -Dsonar.ProjectName=10-Tier \
                        -Dsonar.java.binaries=. \
                        -Dsonar.sources=.'''
                }
            }
        }

        stage('Clean Docker Resources') {
            steps {
                sh '''
                    docker system prune -a -f
                    docker volume prune -f
                '''
            }
        }

        stage('Build and Push to ECR') {
            steps {
                script {
                    // ECR Login
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """

                    // Function to build and push
                    def buildAndPush = { serviceName, dockerfilePath ->
                        def cleanRepoName = serviceName.replaceAll('/', '-')
                        
                        dir("/var/lib/jenkins/workspace/10-tier-app/src/${dockerfilePath}") {
                            sh """
                                aws ecr create-repository --repository-name ${cleanRepoName} --region ${AWS_REGION} || true
                                docker build -t ${ECR_REGISTRY}/${cleanRepoName}:latest .
                                docker push ${ECR_REGISTRY}/${cleanRepoName}:latest
                                docker rmi ${ECR_REGISTRY}/${cleanRepoName}:latest
                            """
                        }
                    }

                    // List of services to build
                    def services = [
                        [name: 'adservice', path: 'adservice'],
                        [name: 'cartservice', path: 'cartservice/src'],
                        [name: 'checkoutservice', path: 'checkoutservice'],
                        [name: 'currencyservice', path: 'currencyservice'],
                        [name: 'emailservice', path: 'emailservice'],
                        [name: 'frontend', path: 'frontend'],
                        [name: 'loadgenerator', path: 'loadgenerator'],
                        [name: 'paymentservice', path: 'paymentservice'],
                        [name: 'productcatalogservice', path: 'productcatalogservice'],
                        [name: 'recommendationservice', path: 'recommendationservice'],
                        [name: 'shippingservice', path: 'shippingservice']
                    ]

                    // Build and push each service
                    services.each { service ->
                        buildAndPush(service.name, service.path)
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withKubeConfig([credentialsId: 'k8-token',
                serverUrl: 'https://E68B349E693BCEC6F716837AE3A0D6CB.gr7.ap-south-1.eks.amazonaws.com',
                clusterName: 'my-eks2']
                ) {
                    sh '''
                        # First clean up existing deployments
                        echo "Cleaning up existing deployments..."
                        kubectl delete deployment --all -n webapps
                        sleep 20

                        # Create namespace if it doesn't exist
                        kubectl create namespace webapps --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Deploy Redis first and wait for it
                        echo "Deploying Redis..."
                        kubectl apply -f k8s-manifestFiles/redis.yaml -n webapps
                        kubectl rollout status deployment/redis -n webapps --timeout=180s
                        
                        # Deploy core services sequentially with waits
                        for service in adservice cartservice checkoutservice currencyservice emailservice paymentservice productcatalogservice recommendationservice shippingservice; do
                            echo "Deploying $service..."
                            kubectl apply -f k8s-manifestFiles/$service.yaml -n webapps
                            kubectl rollout status deployment/$service -n webapps --timeout=180s || true
                            sleep 10
                        done
                        
                        # Deploy frontend
                        echo "Deploying frontend..."
                        kubectl apply -f k8s-manifestFiles/frontend.yaml -n webapps
                        kubectl rollout status deployment/frontend -n webapps --timeout=180s || true
                        sleep 10
                        
                        # Finally deploy loadgenerator
                        echo "Deploying loadgenerator..."
                        kubectl apply -f k8s-manifestFiles/loadgenerator.yaml -n webapps
                        kubectl rollout status deployment/loadgenerator -n webapps --timeout=180s || true
                        
                        # Show final status
                        echo "Deployment Status:"
                        kubectl get deployments -n webapps
                        
                        echo "Pod Status:"
                        kubectl get pods -n webapps
                        
                        echo "Service Status:"
                        kubectl get svc -n webapps
                        
                        # Get detailed status of any failed pods
                        echo "Detailed status of failed pods (if any):"
                        kubectl get pods -n webapps | grep -v Running | grep -v Completed | awk '{print $1}' | xargs -I {} kubectl describe pod {} -n webapps || true
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            sh '''
                echo "Cleaning up Docker resources..."
                docker system prune -a -f
                docker image prune -a -f
                docker volume prune -f
            '''
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
