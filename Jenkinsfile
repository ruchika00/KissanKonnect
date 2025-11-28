pipeline {
    agent {
        kubernetes {
            label 'docker-agent'
            defaultContainer 'jnlp'
            yaml """
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
    - mountPath: /var/run/docker.sock
      name: dockersock
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: /home/jenkins/agent
      name: workspace-volume
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: /home/jenkins/agent
      name: workspace-volume
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
  - name: workspace-volume
    emptyDir: {}
"""
        }
    }
    environment {
        NEXUS_REGISTRY = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"
        NEXUS_USERNAME = "admin"
        NEXUS_PASSWORD = "Changeme@2025"
        NEXUS_PROJECT_PATH = "2401199-project"  // Replace with your actual Nexus repo path/project
    }
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
        stage('SonarQube Scan') {
            steps {
                container('sonar-scanner') {
                    withCredentials([string(credentialsId: 'sonar-token-id', variable: 'SONAR_TOKEN')]) {
                        sh """
                        sonar-scanner \
                        -Dsonar.projectKey=kissankonnect \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                        -Dsonar.token=$SONAR_TOKEN
                        """
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh '''
                    echo "Building Docker Image..."
                    docker build -t kissankonnect:latest .
                    docker image ls
                    '''
                }
            }
        }
        stage('Check Nexus DNS') {
            steps {
                container('kubectl') {
                    sh 'nslookup ${NEXUS_REGISTRY}'
                }
            }
        }
        stage('Login to Docker Registry') {
            steps {
                container('docker') {
                    sh 'docker --version'
                    // Use --password-stdin for secure login
                    sh "echo '${NEXUS_PASSWORD}' | docker login ${NEXUS_REGISTRY} -u ${NEXUS_USERNAME} --password-stdin"
                }
            }
        }
        stage('Build - Tag - Push') {
            steps {
                container('docker') {
                    sh """
                    docker tag kissankonnect:latest ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/kissankonnect:latest
                    docker push ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/kissankonnect:latest
                    docker pull ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/kissankonnect:latest
                    docker image ls
                    """
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    // Your kubectl deployment commands here
                    sh 'kubectl apply -f k8s-deployment.yaml'
                }
            }
        }
    }
}





