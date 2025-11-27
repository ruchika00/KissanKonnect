pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "your_dockerhub_username"    // CHANGE THIS
        IMAGE_NAME = "kissankonnect"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/ruchika00/KissanKonnect.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t $DOCKERHUB_USER/$IMAGE_NAME:latest .
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-pass', variable: 'PASSWORD')]) {
                    sh """
                        echo $PASSWORD | docker login -u $DOCKERHUB_USER --password-stdin
                        docker push $DOCKERHUB_USER/$IMAGE_NAME:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-cred']) {
                    sh """
                        kubectl apply -f K8s_deployment.yaml
                        kubectl apply -f K8s_mysql.yaml
                    """
                }
            }
        }

    }
}

