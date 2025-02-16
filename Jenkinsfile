pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_ACCOUNT_ID = "084828580507"
        AWS_REGION = "ap-south-1"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('git checkout') {
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
                        // Clean repository name - remove slashes and replace with hyphens
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
                withKubeConfig([credentialsId: 'k8-token']) {
                    sh '''
                        # Apply namespace first
                        echo "Creating namespace..."
                        kubectl apply -f k8s-manifestFiles/namespace.yaml
                        
                        # Wait for namespace to be active
                        kubectl wait --for=condition=Active namespace/microservices --timeout=30s
                        
                        # Apply deployments
                        echo "Applying deployments..."
                        kubectl apply -R -f k8s-manifestFiles/deployments/
                        
                        # Apply services
                        echo "Applying services..."
                        kubectl apply -R -f k8s-manifestFiles/services/
                        
                        # Wait for deployments to be ready
                        echo "Waiting for deployments to be ready..."
                        kubectl wait --for=condition=available --timeout=300s -n microservices deployment --all
                        
                        # Show deployment status
                        echo "Deployment Status:"
                        kubectl get deployments -n microservices
                        
                        # Show pods status
                        echo "Pod Status:"
                        kubectl get pods -n microservices
                        
                        # Show services
                        echo "Service Status:"
                        kubectl get svc -n microservices
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
