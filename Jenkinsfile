pipeline {
    agent any

    tools {
        maven 'maven' // Ensure this matches the Maven installation name in Jenkins
    }

    environment {
        ImageName = 'my-app-image'
        BUILD_TAG = "latest"
    }

    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'main', url: 'https://github.com/ariges/SprintbootAPP.git'
            }
        }

        
        stage('Maven Package') {
            steps {
                echo 'Packaging the project...'
                sh 'mvn package'
            }
        }

       stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonarserver') {
            sh '''
                sonar-scanner \
                -Dsonar.projectName=springboot \
                -Dsonar.projectKey=sonarproject4567_springboot \
                -Dsonar.sources=.
            '''
        }
    }
}
        
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh '''
                    docker build -t ${ImageName}:${BUILD_TAG} .
                    docker tag ${ImageName}:${BUILD_TAG} luckyregistry.azurecr.io/${ImageName}:${BUILD_TAG}
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                echo 'Running Trivy scan...'
                sh '''
                    trivy image --format table --severity HIGH,CRITICAL \
                        --output trivy-report.txt luckyregistry.azurecr.io/${ImageName}:${BUILD_TAG}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt'
                }
            }
        }

        stage('Login to ACR and Push Image') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD'),
                    string(credentialsId: 'azure-tenant', variable: 'TENANT_ID')
                ]) {
                    script {
                        echo "Logging into Azure Container Registry..."
                        sh '''
                            az login --service-principal -u "$AZURE_USERNAME" -p "$AZURE_PASSWORD" --tenant "$TENANT_ID"
                            az acr login --name luckyregistry
                            docker push luckyregistry.azurecr.io/${ImageName}:${BUILD_TAG}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying to Kubernetes...'
                    sh '''
                        az aks get-credentials --resource-group LUCKY --name demo-aks
                        kubectl apply -f k8s/petclinic.yml
                        kubectl get all
                    '''
                }
            }
        }
    }
}
