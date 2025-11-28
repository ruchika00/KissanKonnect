pipeline {
    agent {
        kubernetes {
            label 'docker-agent'
            defaultContainer 'docker'
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

  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command:
    - cat
    tty: true

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
        SONAR_HOST_URL = "http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/ruchika00/KissanKonnect.git'
            }
        }

        stage('SonarQube Scan') {
            steps {
                container('sonar-scanner') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=kissankonnect \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh '''
                        echo "Building Docker Image..."
                        docker build -t ${IMAGE_NAME}:latest .
                        docker image ls
                    '''
                }
            }
        }

        stage('Check Nexus DNS') {
            steps {
                container('docker') {
                    sh '''
                        echo "Checking Nexus DNS resolution..."
                        nslookup ${NEXUS_REGISTRY%:*} || true
                    '''
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                container('docker') {
                    sh 'docker --version'
                    sh 'sleep 10'
                    sh 'docker login ${NEXUS_REGISTRY} -u admin -p Changeme@2025'
                }
            }
        }

        stage('Build - Tag - Push') {
            steps {
                container('docker') {
                    sh '''
                        docker tag ${IMAGE_NAME}:latest ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest
                        docker push ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest
                        docker pull ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest
                        docker image ls
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
                        echo "Applying Kubernetes Deployment..."
                        kubectl apply -f K8s_deployment/deployment.yaml

                        echo "Waiting for rollout..."
                        kubectl rollout status deployment/kissankonnect-deployment -n default || true
                    '''
                }
            }
        }
    }
}




