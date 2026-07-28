## Minio (S3) 




```
kubectl label --overwrite ns minio-dev pod-security.kubernetes.io/enforce=privileged
```

```
kubectl patch storageclass vsan-default-storage-policy -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```



```
helm repo add minio https://charts.min.io/
helm repo update
```
```
# minio-values.yaml

# Root credentials for the MinIO Console and S3 API
rootUser: "your-admin-username"
rootPassword: "your-secure-password"

# Set deployment mode: standalone (single node) or distributed (multi-node)
mode: standalone

# Configure data persistence
persistence:
  enabled: true
  size: 10Gi

# Define S3 buckets to create automatically after initialization
buckets:
  - name: "my-s3-bucket"
    policy: "none"
    purge: false

app:
  kubernetes.io
    managed-by=Helm
meta:
  helm.sh
    release-name=minio



```
```
helm install minio minio/minio –namespace minio-dev -f minio-values.yaml
```



```
helm install minio minio/minio \
  --namespace minio-dev \
  --set mode=standalone \
  --set replicas=1 \
  --set persistence.enabled=false \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin123 \
  --set resources.requests.memory=64Mi \
  --set resources.requests.cpu=50m \
  --set resources.limits.memory=256Mi \
  --set resources.limits.cpu=250m \
  --set metrics.prometheus.enabled=false \
  --set service.type=LoadBalancer \
  --set consoleService.type=LoadBalancer

```

```
helm uninstall minio
```

```
apiVersion: v1
kind: Namespace
metadata:
  name: minio-dev
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio-dev
spec:
  storageClassName: development
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: quay.io/minio/minio:latest
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          env:
            - name: MINIO_ROOT_USER
              value: "minioadmin"
            - name: MINIO_ROOT_PASSWORD
              value: "minioadmin123" # Change this for security
          ports:
            - containerPort: 9000
              name: api
            - containerPort: 9001
              name: console
          volumeMounts:
            - name: storage
              mountPath: /data
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: minio-dev
spec:
  type: ClusterIP
  selector:
    app: minio
  ports:
    - name: api
      protocol: TCP
      port: 9000
      targetPort: 9000
    - name: console
      protocol: TCP
      port: 9001
      targetPort: 9001
```

