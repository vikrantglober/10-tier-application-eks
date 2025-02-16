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
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=10-Tier -Dsonar.ProjectName=10-Tier -Dsonar.java.binaries=. -Dsonar.sources=.'''
                }
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

                    // Build and push each service with correct paths
                    buildAndPush('adservice', 'adservice')
                    buildAndPush('cartservice', 'cartservice/src')
                    buildAndPush('checkoutservice', 'checkoutservice')
                    buildAndPush('currencyservice', 'currencyservice')
                    buildAndPush('emailservice', 'emailservice')
                    buildAndPush('frontend', 'frontend')
                    buildAndPush('loadgenerator', 'loadgenerator')
                    buildAndPush('paymentservice', 'paymentservice')
                    buildAndPush('productcatalogservice', 'productcatalogservice')
                    buildAndPush('recommendationservice', 'recommendationservice')
                    buildAndPush('shippingservice', 'shippingservice')
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withKubeConfig(caCertificate: '', 
                             clusterName: 'my-eks2', 
                             contextName: '',
                             credentialsId: 'k8-token', 
                             namespace: 'webapps', 
                             restrictKubeConfigAccess: false,
                             serverUrl: 'https://EBCE08CF45C3AA5A574E126370E5D4FC.gr7.ap-south-1.eks.amazonaws.com') {
                    sh '''
                        kubectl apply -f kubernetes/namespace.yaml
                        kubectl apply -f kubernetes/
                        kubectl get pods -n webapps
                        kubectl get svc -n webapps
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
