pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('git checkout') {
            steps {
                git branch: 'latest', url: 'https://github.com/vikrantglober/10-tier-application-eks.git'
            }
        }
        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=10-Tier -Dsonar.ProjectName=10-Tier -Dsonar.java.binaries=.'''
                }
            }
        }

        def buildAndPush = { serviceName ->
            script {
                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                    dir("/var/lib/jenkins/workspace/10-Tier/src/${serviceName}") {
                        sh "docker build -t mudit097/${serviceName}:latest ."
                        sh "docker push mudit097/${serviceName}:latest"
                        sh "docker rmi mudit097/${serviceName}:latest"
                    }
                }
            }
        }

        stage('adservice') { buildAndPush('adservice') }
        stage('cartservice') { buildAndPush('cartservice/src/') }
        stage('checkoutservice') { buildAndPush('checkoutservice') }
        stage('currencyservice') { buildAndPush('currencyservice') }
        stage('emailservice') { buildAndPush('emailservice') }
        stage('frontend') { buildAndPush('frontend') }
        stage('loadgenerator') { buildAndPush('loadgenerator') }
        stage('paymentservice') { buildAndPush('paymentservice') }
        stage('productcatalogservice') { buildAndPush('productcatalogservice') }
        stage('recommendationservice') { buildAndPush('recommendationservice') }
        stage('shippingservice') { buildAndPush('shippingservice') }

        stage('K8-Deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'my-eks2', contextName: '',
                        credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false,
                        serverUrl: 'https://EBCE08CF45C3AA5A574E126370E5D4FC.gr7.ap-south-1.eks.amazonaws.com') {
                    sh 'kubectl apply -f deployment-service.yml'
                    sh 'kubectl get pods'
                    sh 'kubectl get svc'
                }
            }
        }
    }
}
