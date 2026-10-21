# Original Write up
```
https://www.vmware.com/docs/vmw-kubernetes-workload-migration-to-vks
https://github.com/vmware/vks-validated-solutions/tree/main/VKS-Migrations
```

# Create 2 clusters(not shown here) / new contexts
```
vcf context create cluster-a --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name cluster-a --workload-cluster-namespace namespace-a

vcf context create cluster-b --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name cluster-b --workload-cluster-namespace namespace-b

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
# or
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'


kubectl describe pv pvc-9e-{long name}

kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeHandle}'  #FCD UUID

kubectl get csidrivers


```

# register in namespace-b on sup cluster
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

# problem (on sup cluster)

```
k get pv | grep 3G
k describe pv {above pvc-long number} | grep namespace

result is the Claim is still with namespace-a

```


# New test !!!!!!!!!!!!!!!!!!!!!!!!!

## Context swap to cluster-a / namespace-a 
```
vcf context create cluster-a --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name cluster-a --workload-cluster-namespace namespace-a

# or

vcf context use cluster-a:cluster-a

```

## Create pvc in guest cluster-a vSphere namespace-a

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-guest-pvc2
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: vsan-default-storage-policy
  resources:
    requests:
      storage: 4Gi
```
```
kubectl get nodes # to make sure we are on cluster-a
kubcectl apply -f ./{above file}
kubectl get pvc
kubectl label pvc { name } orf=testvol4g

```
## Check in vCenter (storage -> vsan vol -> Monitor -> Container vol) (should find 4G vol with label) 

## Create POD to mount vol

```
apiVersion: v1
kind: Namespace
metadata:
  name: orf-ns1
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-debug
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: "/mnt"
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-guest-pvc2
```

## Jump onto the POD to see mount moint and place test file
```
kubectl -n orf-ns1 exec -it busybox-debug -- sh 
cd /mnt
touch myfile-abc
ls
exit
```
## Delete pod to make sure the file system is not in use 
```
kubectl delete -f ./{file name from above}
```
## Change the Reclaim Policy to Retain

```
kubectl get pv
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
# check that it is set to Retain! Very important!
kubectl get pv
```

## Still in/on cluster-a / namespace-a 
```
kubectl get pvc
kubectl delete pvc my-guest-pvc2
kubectl get pvc
#The PV status will change to Released. Now, you must strip the old namespace bindings from the PV so a new claim can grab it. 
```
## On Sup cluster (on sup VM!) (/usr/lib/vmware-wcp/decryptK8Pwd.py)
```
kubectl get pv
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl get pv

kubectl get pvc -n namespace-1
kubectl delete pvc { long name from above command} -n namespace-a

kubectl get pv  # should show released

kubectl edit pv <your-pv-name>

#In the YAML editor, look under spec.claimRef.
#Remove the uid line entirely (Spec)(there are 2 uid entries the spec one needs to go, and change the namespace field to match your target namespace. Save and close.
```

# Create the New PVC in the Target Namespace (on sup VM)

Create a manifest for your new PVC in the new namespace. To ensure it binds directly to your existing VMware PV, explicitly specify the volumeName in the spec:
yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: destination-pvc-name
  namespace: namespace-b
spec:
  storageClassName: vsan-default-storage-policy
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4G
  volumeName: <your-pv-name> # Forces binding to the existing PV
```

```
kubectl apply -f ./{file name} -n nasmespace-b
kubect get pvc 
```
Apply this YAML. The new PVC and the existing PV should now change status to Bound. Well no it is pending

## Finalize the Binding (Crucial for Tanzu/vSphere CNS)
```
To ensure the VMware Cloud Native Storage control plane recognizes the new relationship, get the UID of the new PVC and patch it back onto the PV.
1. Get the new PVC's UID:bash
kubectl get pvc destination-pvc-name -n namespace-b -o jsonpath='{.metadata.uid}'
```
## Patch the PV with the new UID:bash
```
kubectl patch pv <your-pv-name> -p '{"spec":{"claimRef":{"uid":"<NEW_PVC_UID>"}}}'
```
### (Optional) Change the persistentVolumeReclaimPolicy back to Delete if you want future automated teardowns to clean up the vSphere datastore.


