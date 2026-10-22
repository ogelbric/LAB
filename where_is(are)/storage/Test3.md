# Using this write up as a base
```
https://github.com/vmware/vks-validated-solutions/blob/main/VKS-Migrations/VKS-to-VKS/examples/2-same-supervisor-different-namespace.md
```


# Set up all the contexts

```
#
#save off the old .kube/config file
#
mv /${HOME}/.kube/config /${HOME}/.kube/config.bak
rm /${HOME}/.kube/config
#
export SUP_CONTEXT="sup3"
export SRC_CONTEXT="cluster-a3"
export DST_CONTEXT="cluster-b3"
#
export SRC_CLUSTER="cluster-a"
export DST_CLUSTER="cluster-b"
export SRC_VSPHERE_NS="namespace-a"
export DST_VSPHERE_NS="namespace-b"


#
kubectl vsphere login --server=10.1.8.132 --vsphere-username administrator@WLD.SSO --insecure-skip-tls-verify
cp /${HOME}/.kube/config "$HOME/supervisor-kubeconfig"
export SUPERVISOR_KUBECONFIG="$HOME/supervisor-kubeconfig"
kubectl get nodes
rm /${HOME}/.kube/config
#
kubectl vsphere login --server=10.1.8.132 --vsphere-username administrator@WLD.SSO --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace ${SRC_VSPHERE_NS} --tanzu-kubernetes-cluster-name ${SRC_CLUSTER} --insecure-skip-tls-verify
cp /${HOME}/.kube/config "$HOME/source-vks-kubeconfig"
export SRC_VKS_KUBECONFIG="$HOME/source-vks-kubeconfig"
kubectl get nodes
rm /${HOME}/.kube/config
#
kubectl vsphere login --server=10.1.8.132 --vsphere-username administrator@WLD.SSO --insecure-skip-tls-verify --tanzu-kubernetes-cluster-namespace ${DST_VSPHERE_NS} --tanzu-kubernetes-cluster-name ${DST_CLUSTER} --insecure-skip-tls-verify
cp /${HOME}/.kube/config "$HOME/destination-vks-kubeconfig"
export DST_VKS_KUBECONFIG="$HOME/destination-vks-kubeconfig"
kubectl get nodes
rm /${HOME}/.kube/config
#
# Alias setup for same SUPERVISOR!
#
alias vks-src='kubectl --kubeconfig=${SRC_VKS_KUBECONFIG}'
alias vks-dst='kubectl --kubeconfig=${DST_VKS_KUBECONFIG}'
alias sup-src='kubectl --kubeconfig=${SUPERVISOR_KUBECONFIG}'
alias sup-dst='kubectl --kubeconfig=${SUPERVISOR_KUBECONFIG}'
#
# Source and destination VKS clusters use different namespaces on the same Supervisor.
export SRC_SUP_NS=${SRC_VSPHERE_NS}
export DST_SUP_NS=${DST_VSPHERE_NS}
#
export SRC_VKS_PVC=my-test-pvc
export DST_VKS_PV=my-test-pv-migrated
export DST_VKS_PVC=my-test-pvc-migrated
#
export DST_SUP_PVC=my-test-pvc-migrated-supervisor
export REGISTER_NAME=same-vcenter-fcd-migration
#
export BACKUP_NAME="vks-migration-${SRC_VKS_NS}"
export RESTORE_NAME="vks-migration-${DST_VKS_NS}"
```

# Confirm that each context points to the expected cluster before continuing:

```
vks-src get nodes
sup-src get nodes
vks-dst get nodes
sup-dst get nodes
```

# Make sure there is a default storage class
```
vks-src -n "$SRC_VKS_NS" patch storageclass vsan-default-storage-policy -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```


# Create a test volume and write identifiable data

```
vks-src -n "$SRC_VKS_NS" apply -f - <<EOF_PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${SRC_VKS_PVC}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF_PVC

vks-src -n "$SRC_VKS_NS" wait \
  --for=jsonpath='{.status.phase}'=Bound \
  "pvc/${SRC_VKS_PVC}" \
  --timeout=5m

vks-src -n "$SRC_VKS_NS" apply -f - <<EOF_POD
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


vks-src -n "$SRC_VKS_NS" wait \
  --for=condition=Ready \
  pod/migration-test-writer \
  --timeout=5m

vks-src -n "$SRC_VKS_NS" exec migration-test-writer -- \
  cat /data/migration-test.txt

```

# Discover and record the storage chain

```
export SRC_VKS_PV=$(
  vks-src -n "$SRC_VKS_NS" get pvc "$SRC_VKS_PVC" \
    -o jsonpath='{.spec.volumeName}'
)

export VKS_PVC_SIZE=$(
  vks-src get pv "$SRC_VKS_PV" \
    -o jsonpath='{.spec.capacity.storage}'
)

export SUP_PVC=$(
  vks-src get pv "$SRC_VKS_PV" \
    -o jsonpath='{.spec.csi.volumeHandle}'
)

export SUP_PV=$(
  sup-src -n "$SRC_SUP_NS" get pvc "$SUP_PVC" \
    -o jsonpath='{.spec.volumeName}'
)

export FCD_UUID=$(
  sup-src get pv "$SUP_PV" \
    -o jsonpath='{.spec.csi.volumeHandle}'
)

export SUP_STORAGE_CLASS=$(
  sup-src -n "$SRC_SUP_NS" get pvc "$SUP_PVC" \
    -o jsonpath='{.spec.storageClassName}'
)

export SUP_ACCESS_MODE=$(
  sup-src -n "$SRC_SUP_NS" get pvc "$SUP_PVC" \
    -o jsonpath='{.spec.accessModes[0]}'
)

printf '%-22s %s\n' \
  'VKS PVC:' "$SRC_VKS_NS/$SRC_VKS_PVC" \
  'VKS PV:' "$SRC_VKS_PV" \
  'PVC size:' "$VKS_PVC_SIZE" \
  'Supervisor PVC:' "$SRC_SUP_NS/$SUP_PVC" \
  'Supervisor PV:' "$SUP_PV" \
  'Supervisor policy:' "$SUP_STORAGE_CLASS" \
  'FCD UUID:' "$FCD_UUID"

```

# More discovery

```
vks-src get pv "$SRC_VKS_PV" \
  -o custom-columns=NAME:.metadata.name,HANDLE:.spec.csi.volumeHandle,RECLAIM:.spec.persistentVolumeReclaimPolicy

sup-src -n "$SRC_SUP_NS" get pvc "$SUP_PVC" \
  -o custom-columns=NAME:.metadata.name,PV:.spec.volumeName,PHASE:.status.phase

sup-src get pv "$SUP_PV" \
  -o custom-columns=NAME:.metadata.name,HANDLE:.spec.csi.volumeHandle,RECLAIM:.spec.persistentVolumeReclaimPolicy

```



# For later to figgure out on how to do this with VCF commands
```
#
vcf context create ${SUP_CONTEXT} --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic
vcf context use ${SUP_CONTEXT}
kubectl get nodes
cp /${HOME}/.kube/config "$HOME/supervisor-kubeconfig"
export SUPERVISOR_KUBECONFIG="$HOME/supervisor-kubeconfig"
#rm /${HOME}/.kube/config
#
vcf context create ${SRC_CONTEXT} --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name ${SRC_CLUSTER} --workload-cluster-namespace ${SRC_VSPHERE_NS}
cp /${HOME}/.kube/config "$HOME/source-vks-kubeconfig"
export SRC_VKS_KUBECONFIG="$HOME/source-vks-kubeconfig"
vcf context use ${SRC_CONTEXT}:${SRC_CLUSTER}
kubectl get nodes
#rm /${HOME}/.kube/config
#
vcf context create ${DST_CONTEXT} --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name ${DST_CLUSTER} --workload-cluster-namespace ${DST_VSPHERE_NS}
cp /${HOME}/.kube/config "$HOME/destination-vks-kubeconfig"
export DST_VKS_KUBECONFIG="$HOME/destination-vks-kubeconfig"
vcf context use ${DST_CONTEXT}:${DST_CLUSTER}
kubectl get nodes
#rm /${HOME}/.kube/config
```
# And option to explore
```
vcf cluster kubeconfig get ${SRC_CLUSTER} --export-file a
```


