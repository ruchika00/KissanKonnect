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
  - name: dind
    image: docker:dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    command:
    - dockerd-entrypoint.sh
    args:
    - --host=tcp://0.0.0.0:2375
    - --host=unix:///var/run/docker.sock
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  - name: jnlp
    image: jenkins/inbound-agent:latest
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent
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
                container('dind') {
                    sh '''
                        echo "Building Docker image for PHP project..."
                        docker build -t ${IMAGE_NAME}:latest .
                        docker image ls
                    '''
                }
            }
        }

        stage('Login to Nexus Docker Registry') {
            steps {
                container('dind') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-docker-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
                            echo "Logging in to Nexus Docker Registry..."
                            echo "${NEXUS_PASS}" | docker login ${NEXUS_REGISTRY} -u "${NEXUS_USER}" --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Tag and Push Docker Image') {
            steps {
                container('dind') {
                    sh '''
                        echo "Tagging image with build number and latest tag..."
                        docker tag ${IMAGE_NAME}:latest ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:${BUILD_NUMBER}
                        docker tag ${IMAGE_NAME}:latest ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest

                        echo "Pushing both tags to Nexus registry..."
                        docker push ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Create Kubernetes Namespace and Secret') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl get namespace 2401199 || kubectl create namespace 2401199

                        kubectl create secret docker-registry nexus-secret \
                          --docker-server=${NEXUS_REGISTRY} \
                          --docker-username=${NEXUS_USER} \
                          --docker-password=${NEXUS_PASS} \
                          --namespace=2401199 \
                          --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    dir('K8s_deployment') {
                        sh '''
                            # Replace image tag in deployment.yaml with the new build number
                            sed -i "s|image: .*/${IMAGE_NAME}:.*|image: ${NEXUS_REGISTRY}/${NEXUS_PROJECT_PATH}/${IMAGE_NAME}:${BUILD_NUMBER}|g" deployment.yaml

                            kubectl apply -f deployment.yaml -n 2401199

                            kubectl rollout status deployment/kissankonnect-deployment -n 2401199
                        '''
                    }
                }
            }
        }
    }
}

