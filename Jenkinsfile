pipeline {
    agent {
        kubernetes {
            // Define the custom Pod with all necessary containers (dind, kubectl, sonar-scanner)
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command:
    - cat
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
    securityContext:
      runAsUser: 0
      readOnlyRootFilesystem: false
    env:
    - name: KUBECONFIG
      value: /kube/config
    volumeMounts:
    - name: kubeconfig-secret
      mountPath: /kube/config
      subPath: kubeconfig
  - name: dind
    image: docker:dind
    securityContext:
      privileged: true  # Needed to run Docker daemon
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""  # Disable TLS for simplicity
    volumeMounts:
    - name: docker-config
      mountPath: /etc/docker/daemon.json
      subPath: daemon.json  # Mount the file directly here
  volumes:
  - name: docker-config
    configMap:
      name: docker-daemon-config
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret
'''
        }
    }

    stages {
        // --- 1. Build Docker Image (Using dind container) ---
        stage('Build Docker Image') {
            steps {
                container('dind') {
                    sh '''
                        sleep 15
                        # Build the Docker image for your PHP application
                        docker build -t kissankonnect:latest .
                        docker image ls
                    '''
                }
            }
        }
        
        // --- 2. Run Tests in Docker (Assuming PHPUnit, coverage XML output) ---
        stage('Run Unit Tests (PHP)') {
            steps {
                container('dind') {
                    sh '''
                        # Run tests inside the built image container
                        # This command assumes PHPUnit is installed and configured to output a coverage.xml
                        # You may need to adjust this command based on your specific PHP setup (e.g., entrypoint/command)
                        docker run --rm -v $(pwd):/app -w /app kissankonnect:latest \
                        vendor/bin/phpunit --coverage-clover coverage.xml
                    '''
                }
            }
        }
        
        // --- 3. SonarQube Analysis (Using sonar-scanner container) ---
        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                     // NOTE: Update 'sonar-token-2401199' to your specific credentialsId if different
                    withCredentials([string(credentialsId: 'sonar-token-2401199', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=2401152_KissanKonnect \
                                -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                                -Dsonar.login=sqp_4253752f0458c40c65e583bde12308d9fb61bba9
                                -Dsonar.sources=. \
                                -Dsonar.language=php \
                                -Dsonar.php.coverage.reportPaths=coverage.xml \
                                -Dsonar.projectName=KissanKonnect-2401152
                        '''
                    }
                }
            }
        }
        
        // --- 4. Login to Docker Registry (Using dind container) ---
        stage('Login to Docker Registry') {
            steps {
                container('dind') {
                    sh 'docker --version'
                    sh 'sleep 10'
                    // NOTE: Update the registry URL and credentials if they are different
                    sh 'docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 -u admin -p Changeme@2025'
                }
            }
        }
        
        // --- 5. Tag and Push Image (Using dind container) ---
        stage('Build - Tag - Push') {
            environment {
                // Use your Roll No and Project Name for the tag
                IMAGE_TAG = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401152-project/kissankonnect:latest"
            }
            steps {
                container('dind') {
                    // Tag the locally built image with the full registry path
                    sh "docker tag kissankonnect:latest ${env.IMAGE_TAG}"
                    // Push to the Nexus Docker Registry
                    sh "docker push ${env.IMAGE_TAG}"
                    // Verify the image was pushed (optional: pull it back)
                    sh "docker pull ${env.IMAGE_TAG}"
                    sh 'docker image ls'
                }
            }
        }
        
        // --- 6. Deploy Application to Kubernetes (Using kubectl container) ---
        stage('Deploy KissanKonnect') {
            steps {
                container('kubectl') {
                    script {
                        // Ensure the k8s-deployment directory exists in your repo
                        dir('k8s-deployment') {
                            sh '''
                                # Apply all resources in the deployment YAML
                                # Make sure your YAML uses the image tag from the previous stage
                                kubectl apply -f kissan-konnect-deployment.yaml

                                # Wait for rollout to complete in your namespace (e.g., 2401152)
                                kubectl rollout status deployment/kissan-konnect-deployment -n 2401152
                            '''
                        }
                    }
                }
            }
        }
    }
}
