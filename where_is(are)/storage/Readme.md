# Create pvc in guest cluster-a vSphere namespace-a

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-guest-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: vsan-default-storage-policy
  resources:
    requests:
      storage: 3Gi
```

