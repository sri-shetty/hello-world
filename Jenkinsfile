pipeline {
    agent any
    // agent { dockerfile true }

    tools {
        // Use the JDK configured in Global Tool Configuration
        jdk 'jdk21'
    }

    environment {
        // Docker command
        PATH = "/opt/homebrew/bin:${env.PATH}"
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

        stage('Debug Environment') {
            steps {
                sh 'echo "JAVA_HOME is set to: $JAVA_HOME"'
                sh 'java -version'
                sh 'echo "Current PATH: $PATH"'
                sh 'which docker'
                sh 'docker --version'
            }
        }

        stage('Build Application') {
            steps {
                sh './mvnw clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def customImage = docker.build("${DOCKER_IMAGE}")
                    //echo "Image build moved to next stage"
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                //script {
                //    try {
                        // docker.withRegistry("https://${ACR_LOGIN_SERVER}", "${DOCKER_CREDENTIALS_ID}") {
                            // customImage.push()
                            // customImage.push('latest')
                            // docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push()
                            // docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push('latest')
                            // sh "/opt/homebrew/bin/docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                            // sh "/opt/homebrew/bin/docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}:latest"
                        //}
                //        echo 'Docker image pushed successfully to ACR'
                //    }
                //    catch (Exception e) {
                //        echo "Failed to push Docker image to ACR: ${e}"
                //        currentBuild.result = 'FAILURE'
                //        error('Docker push failed.')
                //    }

                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh """
                        docker login ${ACR_LOGIN_SERVER} -u ${USERNAME} -p ${PASSWORD}
                        docker push ${DOCKER_IMAGE}
                    """

                }
            }
        }

        stage('Set to AKS') {
            steps {
                withCredentials([string(credentialsId: "${AZURE_CREDENTIALS_ID}", variable: 'AZURE_CREDENTIALS_JSON')]) {
                    sh 'echo $AZURE_CREDENTIALS_JSON > azure_credentials.json'
                }

                sh '''
                az login --service-principal -u $(jq -r .clientId azure_credentials.json) -p $(jq -r .clientSecret azure_credentials.json) --tenant $(jq -r .tenantId azure_credentials.json)
                az aks get-credentials --resource-group sriResourceGroup --name sriAKSCluster
                '''

                sh 'rm azure_credentials.json'
            }
        }

        stage('Deploy Pod') {
            steps {
                //withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                    """
                //}
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