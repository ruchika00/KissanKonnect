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
    image: docker:24.0
    command: ["cat"]
    tty: true
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: dockersock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }

    environment {
        REGISTRY_URL = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"
        REPO_NAME    = "2401152"                       // your roll number repo
        IMAGE_NAME   = "kissankonnect"                 // your project image
        K8S_DEPLOYMENT = "kissankonnect-web"           
        K8S_NAMESPACE = "default"
    }

    stages {

        stage('Clean') {
            steps { cleanWs() }
        }

        stage('Checkout Source Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/ruchika00/KissanKonnect.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    IMAGE_TAG = env.BUILD_NUMBER
                    FULL_IMAGE = "${REGISTRY_URL}/${REPO_NAME}/${IMAGE_NAME}:${IMAGE_TAG}"

                    sh "docker build -t ${FULL_IMAGE} ."
                }
            }
        }

        stage('Login to Nexus Registry') {
            steps {
                script {
                    sh """
                       echo "${NEXUS_PASSWORD}" | docker login ${REGISTRY_URL} \
                       -u ${NEXUS_USERNAME} --password-stdin
                    """
                }
            }
        }

        stage('Push Image to Nexus') {
            steps {
                sh "docker push ${FULL_IMAGE}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        kubectl set image deployment/${K8S_DEPLOYMENT} \
                        kissankonnect-web=${FULL_IMAGE} \
                        -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success { echo "🎉 Deployment successful!" }
        failure { echo "❌ Build failed!" }
    }
}
