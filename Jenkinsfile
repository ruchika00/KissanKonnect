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
    command:
    - cat
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
        REGISTRY_URL = "127.0.0.1:30085"       // Nexus IP with port accessible by Jenkins agent
        REPO_NAME    = "2401152"               // Your Nexus repository name, confirm exact repo name
        IMAGE_NAME   = "kissankonnect"         // Your image name
        K8S_DEPLOYMENT = "kissankonnect-web"  // Kubernetes deployment name
        K8S_NAMESPACE  = "default"             // Namespace where app is deployed
        NEXUS_USERNAME = credentials('nexus-username')  // Store these credentials in Jenkins Credentials Manager
        NEXUS_PASSWORD = credentials('nexus-password')
    }

    stages {
        stage('Clean') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
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

        stage('Push Image') {
            steps {
                script {
                    sh "docker push ${FULL_IMAGE}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        kubectl set image deployment/${K8S_DEPLOYMENT} \
                        web=${FULL_IMAGE} -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Build failed!"
        }
    }
}
