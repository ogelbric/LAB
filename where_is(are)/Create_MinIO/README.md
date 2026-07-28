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

