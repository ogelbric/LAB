# Original Write up
```
https://www.vmware.com/docs/vmw-kubernetes-workload-migration-to-vks
https://github.com/vmware/vks-validated-solutions/tree/main/VKS-Migrations
```

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
  namespace: namespace-b
spec:
  volumeID: "2c5999e4-e4a7-4cf8-9220-ac9f2d04f4b1"
  accessMode: ReadWriteOnce
  pvcName: data-volume-adopted

```



```
kubectl --kubeconfig="${SUPERVISOR_KUBECONFIG}" \
  -n destination-supervisor-namespace get \
  cnsregistervolume/register-application-data \
  -o jsonpath='{.status.error}{"\n"}'

kubectl -n namespace-b get cnsregistervolume/register-application-data -o jsonpath='{.status.error}{"\n"}'

```

# on sup cluster
```
kubectl delete pvc {name} -n namespace-a
```

# In guest cluster-b creae pv
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-existing-pv
  annotations:
    pv.kubernetes.io/provisioned-by: csi.vsphere.vmware.com
spec:
  storageClassName: "vsan-default-storage-policy"
  capacity:
    storage: 3Gi # Match the actual size of your existing disk
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: csi.vsphere.vmware.com
    volumeHandle: "2c5999e4-e4a7-4cf8-9220-ac9f2d04f4b1"
    fsType: ext4
```

# in guest cluster-b create pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-existing-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: "vsan-default-storage-policy"
  resources:
    requests:
      storage: 3Gi
  volumeName: my-existing-pv
```

# problem

```
k get pv | grep 3G
k describe pv {above pvc-long number} | grep namespace

result is the Clain is still with namesoace-a

```

