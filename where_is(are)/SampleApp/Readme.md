# Sample App - nginx

```

apiVersion: v1
kind: Namespace
metadata:
  name: orf-web-apps
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-orf
  namespace: orf-web-apps
  labels:
    app: nginx-orf
    tier: frontend
    env: prod
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: nginx-orf
      tier: frontend
      env: prod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 15%
      maxUnavailable: 15%
  template:
    metadata:
      labels:
        app: nginx-orf
        tier: frontend
        env: prod
    spec:
      containers:
        - name: nginx
          #image: nginx:latest
          image: gcr.io/boreal-rain-256712/nginx
          ports:
            - containerPort: 80
          # 1. Startup Probe: Protects slow-starting containers by holding off liveness/readiness
          startupProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30 # Gives the app up to 150 seconds to finish starting
          # 2. Liveness Probe: Kills and restarts the container if it stops responding
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          # 3. Readiness Probe: Pulls the Pod out of Service endpoints if traffic shouldn't be sent yet
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 2
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-orf
  namespace: orf-web-apps
  labels:
    app: nginx-orf
    tier: frontend
    env: prod
spec:
  type: LoadBalancer
  selector:
    app: nginx-orf
    tier: frontend
    env: prod
  ports:
    - port: 80
      targetPort: 80


```

