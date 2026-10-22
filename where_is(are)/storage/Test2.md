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


# Not sure about this.. 
```
kubectl edit cnsvolumeinfoes -n vmware-system-csi ba...{long name coming from kubectl get cnsvolumeinfoes and my 3 hour old pv/pvc I created}

```



# Need govc installed
```
sudo curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" > /tmp/a
sudo tar -C /usr/local/bin -xvzf /tmp/a govc
```






