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
vi minio.yaml
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
  storageClassName: vsan-default-storage-policy
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
  type: LoadBalancer
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
## Install the minio mc cli command 

```
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
./mc --help


```

## Create Bucket with mc

```
#!/usr/bin/bash

# Configuration variables
#MINIO_URL="http://localhost:9000"
MINIO_URL="http://`kubectl get svc -n minio-dev | tail -1 | awk '{print $4}'`:9000"
ACCESS_KEY="minioadmin"
SECRET_KEY="minioadmin123"
BUCKET_NAME="my-new-bucket77"
ALIAS="s3"
ALIAS2="browser"
API="S3v4"

# Configure the MinIO client alias
./mc alias set $ALIAS $MINIO_URL $ACCESS_KEY $SECRET_KEY --api $API

# Create the bucket (ignores if it already exists)
./mc mb --ignore-existing $ALIAS2/$BUCKET_NAME

echo "Bucket '$BUCKET_NAME' successfully verified/created."

```







