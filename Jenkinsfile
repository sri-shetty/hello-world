pipeline {
    agent any

    tools {
        // Use the JDK configured in Global Tool Configuration
        jdk 'jdk21'
    }

    environment {
        // Azure Container Registry
        ACR_NAME = "sriacrregistry"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        DOCKER_IMAGE = "${ACR_LOGIN_SERVER}/hello-world"

        // Docker Credentials
        DOCKER_CREDENTIALS_ID = "acr-credentials"

        // Azure Service Principal
        AZURE_CREDENTIALS_ID = "azure-sp-credentials"

        // Kubernetes
        KUBECONFIG_CREDENTIAL_ID = "kubeconfig"

        // GitHub Repository
        GIT_REPO = "https://github.com/sri-shetty/hello-world.git"
        GIT_BRANCH = "main"
    }

    stages {
        stage('Checkout Source') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Build Application') {
            steps {
                sh 'JAVA_HOME is ${JAVA_HOME}'
                sh 'java -version'
                sh './mvnw clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    docker.withRegistry("https://${ACR_LOGIN_SERVER}", "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push()
                        docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([string(credentialsId: "${AZURE_CREDENTIALS_ID}", variable: 'AZURE_CREDENTIALS_JSON')]) {
                    sh 'echo $AZURE_CREDENTIALS_JSON > azure_credentials.json'
                }

                sh '''
                az login --service-principal -u $(jq -r .clientId azure_credentials.json) -p $(jq -r .clientSecret azure_credentials.json) --tenant $(jq -r .tenantId azure_credentials.json)
                az aks get-credentials --resource-group sriResourceGroup --name sriAKSCluster
                kubectl set image deployment/hello-world-deployment hello-world=${DOCKER_IMAGE}:${BUILD_NUMBER}
                '''

                sh 'rm azure_credentials.json'
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER}'
                sh 'docker rmi ${DOCKER_IMAGE}:latest'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Deployment Successful!'
        }
        failure {
            echo 'Deployment Failed.'
        }
    }
}