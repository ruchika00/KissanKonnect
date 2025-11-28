# -------------------------------
# PersistentVolumeClaim
# -------------------------------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: recipe-static-pvc
  namespace: "2401063"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recipe-finder-deployment
  namespace: "2401063"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: recipe-finder
  template:
    metadata:
      labels:
        app: recipe-finder
    spec:
      imagePullSecrets:
        - name: nexus-secret
      containers:
        - name: recipe-finder
          image: nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401063/recipe-finder:latest
          imagePullPolicy: Always

          ports:
            - containerPort: 80

          env:
            - name: VITE_API_KEY
              value: "cc61bff714d14ffe98913e198a562acf"

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 30

          volumeMounts:
            - name: recipe-static
              mountPath: /usr/share/nginx/html

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

      volumes:
        - name: recipe-static
          persistentVolumeClaim:
            claimName: recipe-static-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: recipe-finder-service
  namespace: "2401063"
spec:
  selector:
    app: recipe-finder
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: recipe-finder-ingress
  namespace: "2401063"
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: 2401063.imcc.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: recipe-finder-service
                port:
                  number: 80
