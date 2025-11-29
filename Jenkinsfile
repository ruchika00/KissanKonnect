pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/kaniko-project/executor:latest
    args: ["--storage-driver=vfs"]
    volumeMounts:
    - name: kaniko-secret
      mountPath: /kaniko/.docker/
  - name: kubectl
    image: nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/kubectl:1.28
    command: ['cat']
    tty: true
    volumeMounts:
    - name: kubeconfig
      mountPath: /kube
  - name: sonar
    image: nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/alpine:latest
    command: ['cat']
    tty: true
  volumes:
  - name: kaniko-secret
    secret:
      secretName: regcred
  - name: kubeconfig
    secret:
      secretName: kubeconfig-secret
"""
        }
    }

    environment {
        IMAGE = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401152-project/kissankonnect:latest"
    }

    stages {

        stage('SonarQube Analysis') {
            steps {
                container('sonar') {
                    sh """
                        apk add curl unzip openjdk11
                        curl -o scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.zip
                        unzip scanner.zip
                        sonar-scanner-*/bin/sonar-scanner \
                            -Dsonar.projectKey=2401152_KissanKonnect \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                            -Dsonar.login=sqp_4253752f0458c40c65e583bde12308d9fb61bba9
                    """
                }
            }
        }

        stage('Build Image with Kaniko') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                            --context `pwd` \
                            --dockerfile Dockerfile \
                            --destination ${IMAGE}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl --kubeconfig /kube/config apply -f k8s-deployment/kissan-konnect-deployment.yaml
                        kubectl --kubeconfig /kube/config rollout status deployment/kissan-konnect-deployment -n 2401152
                    """
                }
            }
        }
    }
}

