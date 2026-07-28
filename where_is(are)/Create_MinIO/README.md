## Minio (S3) 




```
kubectl label --overwrite ns default pod-security.kubernetes.io/enforce=privileged
```

```
kubectl patch storageclass vsan-default-storage-policy -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```



```
helm repo add minio https://charts.min.io/
helm repo update
```

```
helm install minio minio/minio \
  --namespace minio-system \
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


