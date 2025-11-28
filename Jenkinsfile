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
        NEXUS_REGISTRY = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"
        NEXUS_PROJECT_PATH = "2401199-project"
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
                        docker build -t ${IMAGE_NAME}:latest .
                    """
                }
            }
        }

        stage('Login to Nexus Docker Registry') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-docker-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                            echo "Logging in to Nexus Docker Registry..."
                            echo "${NEXUS_PASS}" | docker login ${NEXUS_REGISTRY} -u "${NEXUS_USER}" --password-stdin
                        """
                    }
                }
            }
        }

        stage('Tag and Push to Nexus') {
            steps {
                container('docker') {
                    sh """
                        echo "Tagging image for Nexus..."
                        docker tag ${IMAGE_NAME}:latest ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest

                        echo "Pushing image to Nexus..."
                        docker push ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest

                        echo "Pulling image back to verify..."
                        docker pull ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest
                    """
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

