pipeline {
    agent {
        kubernetes {
            label 'docker-agent'
            defaultContainer 'jnlp'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:20.10.16
    command:
    - cat
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"

  - name: jnlp
    image: jenkins/inbound-agent:latest
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"

  - name: kubectl
    image: bitnami/kubectl:latest
    command:
      - cat
    tty: true

  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
  - name: workspace-volume
    emptyDir: {}
'''
        }
    }

    environment {
        DOCKERHUB_USER = "ruchika28"
        IMAGE_NAME = "kissankonnect"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/ruchika00/KissanKonnect.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh """
                        echo "Building Docker Image..."
                        docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:latest .
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo "Logging in to DockerHub..."
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin

                            echo "Tagging image..."
                            docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${DOCKER_USER}/${IMAGE_NAME}:latest

                            echo "Pushing image..."
                            docker push ${DOCKER_USER}/${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh """
                        echo "Applying Kubernetes Deployment..."
                        kubectl apply -f K8s_deployment/deployment.yaml

                        echo "Waiting for rollout..."
                        kubectl rollout status deployment/kissankonnect-deployment -n default || true
                    """
                }
            }
        }
    }
}
