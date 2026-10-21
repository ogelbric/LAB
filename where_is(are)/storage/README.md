# Original Write up
```
https://www.vmware.com/docs/vmw-kubernetes-workload-migration-to-vks
https://github.com/vmware/vks-validated-solutions/tree/main/VKS-Migrations
```

# Create 2 new contexts
```
vcf context create  cluster-a --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name cluster-a --workload-cluster-namespace namespace-a

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

result is the Clain is still with namesoace-a

```


## New test 

```

Method 2: The Manual Rebind (Without Copying Data)

If you have a large volume and want to move it without copying data, you can release the underlying vSphere Cloud Native Storage (CNS) disk and rebind it to the new namespace.

Step 1: Change the Reclaim Policy to Retain

By default, deleting a PVC deletes the backing storage. You must change the policy so the vSphere CNS volume stays intact.
bash
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
Use code with caution.

Step 2: Delete the Old PVC & Edit the PV Spec

1. Scale down your application in the old namespace.
2. Delete the old PVC:bash
kubectl delete pvc <old-pvc-name> -n source-ns
Use code with caution.
3. The PV status will change to Released. Now, you must strip the old namespace bindings from the PV so a new claim can grab it. Edit the PV:bash
kubectl edit pv <your-pv-name>
Use code with caution.
4. In the YAML editor, look under spec.claimRef. Remove the uid line entirely, and change the namespace field to match your target namespace. Save and close.

Step 3: Create the New PVC in the Target Namespace

Create a manifest for your new PVC in the new namespace. To ensure it binds directly to your existing VMware PV, explicitly specify the volumeName in the spec:
yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: destination-pvc-name
  namespace: target-ns
spec:
  storageClassName: <your-vsphere-storage-class>
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: <must-match-exact-pv-size>
  volumeName: <your-pv-name> # Forces binding to the existing PV
Use code with caution.
Apply this YAML. The new PVC and the existing PV should now change status to Bound.

Step 4: Finalize the Binding (Crucial for Tanzu/vSphere CNS)

To ensure the VMware Cloud Native Storage control plane recognizes the new relationship, get the UID of the new PVC and patch it back onto the PV.
1. Get the new PVC's UID:bash
kubectl get pvc destination-pvc-name -n target-ns -o jsonpath='{.metadata.uid}'
Use code with caution.
2. Patch the PV with the new UID:bash
kubectl patch pv <your-pv-name> -p '{"spec":{"claimRef":{"uid":"<NEW_PVC_UID>"}}}'
Use code with caution.
3. (Optional) Change the persistentVolumeReclaimPolicy back to Delete if you want future automated teardowns to clean up the vSphere datastore.
```

