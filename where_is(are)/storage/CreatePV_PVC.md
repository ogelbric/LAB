# Sample create application with PV / PVC


## 1 - Supervisor Cluster / vSphere Namespace Login
```
export API_IP="10.1.8.132"
export CL="cluster1"
export SUP_NS="namespace1000"
#
# API end point is supervisor IP cert 
#
openssl s_client -connect ${API_IP}:443 -showcerts </dev/null 2>/dev/null  | openssl x509 -outform PEM > supervisor.crt
#
# Connecting to supervisor cluster
#
vcf context create  mysupervisor01 --endpoint https://${API_IP} --username administrator@WLD.SSO --ca-certificate supervisor.crt --auth-type basic
#
vcf context list
vcf context use      # use the context of the supervisor:vSphereNameSpace
kubectl get nodes
#
vcf clusters list -A
#
# build new cluster (make sure the Harbor cert is in the Supervisor then Harbor cert will end up in the guest cluster!)
#
kubectl apply -f {Sample Cluster yaml}
#sample cluster yaml see below

vcf clusters list -A
# or
kubectl get clusters -A
# get base line PV/PVC's
kubectl get sc
kubectl get pv -A
kubectl get pvc -A

```

## 2 - Guest Cluster Login

```
#
# Connecting to guest cluster
#
vcf context create  myguestcluster01 --endpoint https://${API_IP} --username administrator@WLD.SSO --ca-certificate supervisor.crt --auth-type basic --workload-cluster-name ${CL} --workload-cluster-namespace ${SUP_NS}
vcf context list
vcf context use myguestcluster01:cluster1
kubectl get nodes 
```

## 3 - Apply Application

### Default Storage Class
```
export SRC_VKS_NS=default
export STORE="vsan-default-storage-policy"
export SRC_VKS_PVC=my-test-pvc
#
# Make sure the default storage Class is set
#
kubectl -n "$SRC_VKS_NS" patch storageclass ${STORE} -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```
### PVC
```
kubectl -n "$SRC_VKS_NS" apply -f - <<EOF_PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${SRC_VKS_PVC}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
EOF_PVC
```
#### Test PVC
```
kubectl -n "$SRC_VKS_NS" wait \
  --for=jsonpath='{.status.phase}'=Bound \
  "pvc/${SRC_VKS_PVC}" \
  --timeout=5m
```
### Sample Application
```
kubectl -n "$SRC_VKS_NS" apply -f - <<EOF_POD
apiVersion: v1
kind: Namespace
metadata:
  name: default
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
---
apiVersion: v1
kind: Pod
metadata:
  name: migration-test-writer
spec:
  restartPolicy: Never
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c"]
      args:
        - |
          echo "VKS migration test $(date -u +%FT%TZ)" > /data/migration-test.txt
          sync
          cat /data/migration-test.txt
          sleep 3600
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ${SRC_VKS_PVC}
EOF_POD
```
#### Test Application
```
kubectl -n "$SRC_VKS_NS" wait \
  --for=condition=Ready \
  pod/migration-test-writer \
  --timeout=5m
```
#### Look at the data written to PVC
```
kubectl -n "$SRC_VKS_NS" exec migration-test-writer -- \
  cat /data/migration-test.txt
```


#### Sample Cluster yaml
```
kubectl  apply -f - <<EOF_CLUSTER
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: ${CL}
  namespace: ${SUP_NS}
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
        - 192.168.156.0/20
    services:
      cidrBlocks:
        - 10.96.0.0/12
    serviceDomain: cluster.local
  topology:
    class: builtin-generic-v3.6.0
    classNamespace: vmware-system-vks-public
    version: v1.33.6---vmware.1-fips-vkr.2
    variables:
      - name: vsphereOptions
        value:
          persistentVolumes:
            defaultStorageClass: vsan-default-storage-policy
      - name: kubernetes
        value:
          certificateRotation:
            enabled: true
            renewalDaysBeforeExpiry: 90
      - name: vmClass
        value: best-effort-medium
      - name: storageClass
        value: vsan-default-storage-policy
      - name: volumes
        value:
        - name: containerd
          mountPath: /var/lib/containerd
          storageClass: vsan-default-storage-policy
          capacity: 15Gi
        - name: kubelet-override
          mountPath: /var/lib/kubelet
          storageClass: vsan-default-storage-policy
          capacity: 10Gi          
        - name: log
          mountPath: /var/log
          storageClass: vsan-default-storage-policy
          capacity: 10Gi          
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
        - class: node-pool
          name: ${CL}-worker
          replicas: 1
EOF_CLUSTER
```














```
