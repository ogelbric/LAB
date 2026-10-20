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

# On supervisor cluster check the following (via vCenter to sup cluster (/usr/lib/vmware-wcp/decryptK8Pwd.py))

```
k get pvc -n namespace-a

kubectl get pv
kubectl describe pv pvc-9e-{long name}
kubectl edit pv pvc-9e-{long name}
# change Delete to Retain
kubectl describe pv pvc-9e-{long name}

kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeHandle}'  #FCD UUID

kubectl get csidrivers


```

```
apiVersion: cns.vmware.com/v1alpha1
kind: CnsRegisterVolume
metadata:
  name: register-application-data
  namespace: destination-supervisor-namespace
spec:
  volumeID: "2c5999e4-e4a7-4cf8-9220-ac9f2d04f4b1"
  accessMode: ReadWriteOnce
  pvcName: data-volume-adopted

```



```
kubectl --kubeconfig="${SUPERVISOR_KUBECONFIG}" \
  -n destination-supervisor-namespace wait \
  --for=jsonpath='{.status.registered}'=true \
  cnsregistervolume/register-application-data \
  --timeout=5m
```
