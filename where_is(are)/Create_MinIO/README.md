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

```
#For Ubuntu OS
#
sudo apt install nodejs
sudo apt-get update
sudo apt install npm

node -v && npm --version

npm run start
npm init -y

```

## Create bucket
```
#!/bin/bash
# Usage: ./create-bucket.sh <bucket-name>

BUCKET=$1
ACCESS_KEY="minioadmin"
SECRET_KEY="minioadmin123"
#MINIO_URL="http://localhost:9010" # Use your ClusterIP/Forwarded URL
MINIO_URL="http://`kubectl get svc -n minio-dev | tail -1 | awk '{print $4}'`:9001" # Use your ClusterIP/Forwarded URL

DATE=$(date -R)
SIGNATURE=$(echo -en "PUT\n\n\n${DATE}\n/${BUCKET}" | openssl dgst -sha1 -hmac "${SECRET_KEY}" -binary | base64)

curl -X PUT \
     -H "Host: `kubectl get svc -n minio-dev | tail -1 | awk '{print $4}'`:9001" \
     -H "Date: ${DATE}" \
     -H "Authorization: AWS ${ACCESS_KEY}:${SIGNATURE}" \
     "${MINIO_URL}/${BUCKET}"
```



