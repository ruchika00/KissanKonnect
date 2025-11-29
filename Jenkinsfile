pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: sonar
    image: sonarsource/sonar-scanner-cli
    command: ["cat"]
    tty: true

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true

  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args: ["--dockerfile=Dockerfile", "--context=/workspace", "--destination=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401152-project/kissankonnect:latest", "--skip-tls-verify"]
    volumeMounts:
    - name: kaniko-secret
      mountPath: /kaniko/.docker
  volumes:
  - name: kaniko-secret
    secret:
      secretName: nexus-docker-secret   # <-- MUST exist in your namespace
'''
        }
    }

    stages {

        stage('Move Code to Kaniko Workspace') {
            steps {
                container('kaniko') {
                    sh 'cp -r $PWD /workspace'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar') {
                    withCredentials([string(credentialsId: 'sonar-token-2401199', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=2401152_KissanKonnect \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                              -Dsonar.login=$SONAR_TOKEN \
                              -Dsonar.php.coverage.reportPaths=coverage.xml
                        '''
                    }
                }
            }
        }

        stage('Build & Push Image to Nexus (Kaniko)') {
            steps {
                container('kaniko') {
                    sh '''
                        echo "Building & pushing image via Kaniko..."
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl apply -f k8s-deployment/kissan-konnect-deployment.yaml
                        kubectl rollout status deployment/kissan-konnect-deployment -n 2401152
                    '''
                }
            }
        }
    }
}
