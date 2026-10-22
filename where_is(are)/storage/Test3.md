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

# kill the busybix POD

```
vks-src -n "$SRC_VKS_NS" delete pod migration-test-writer \
  --wait=true
```
# Check for remaining VKS VolumeAttachment objects:
```
vks-src get volumeattachments \
  -o custom-columns=NAME:.metadata.name,PV:.spec.source.persistentVolumeName,NODE:.spec.nodeName \
  | grep -F "$SRC_VKS_PV" || true
```

# Do not continue while the source volume remains attached to a running workload.


# govc volume.rm -keep requires govc 0.49.0 or later:
# Install govc
```
sudo curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" > /tmp/a
sudo tar -C /usr/local/bin -xvzf /tmp/a govc
```

```
govc version
govc volume.rm -h | grep -F -- '-keep'
```

# Configure the normal GOVC_URL, GOVC_USERNAME, GOVC_PASSWORD, GOVC_DATASTORE and inventory defaults for the vCenter hosting the source FCD.

```
export GOVC_URL="vc-wld01-a.site-a.vcf.lab"
export GOVC_USERNAME="administrator@WLD.SSO"
export GOVC_PASSWORD='VMware123!VMware123!'
export GOVC_DATASTORE="vsan-wld01-01a"
export GOVC_INSECURE="true"
export GOVC_DATACENTER="dc-a"
export GOVC_NETWORK="mgmt-vds01-wld01-01a"
# Test the login
govc volume.ls -L "$FCD_UUID"

export FCD_BACKING_ID=$(govc volume.ls -L "$FCD_UUID")

printf '%-20s %s\n' \
  'CNS volume ID:' "$FCD_UUID" \
  'FCD backing ID:' "$FCD_BACKING_ID"

govc volume.ls -l "$FCD_UUID"
govc disk.ls -l "$FCD_BACKING_ID"
```


# Create a powered-off helper VM and attach the FCD
# The helper VM is a safety interlock only. It must remain powered off and no guest operating system must mount or modify the disk.

# Discover the backing path:

```
export FCD_PATH=$(
  govc disk.ls -json "$FCD_BACKING_ID" |
    jq -er '.objects[0].config.backing.filePath'
)

export HELPER_VM="${FCD_UUID}-migration-guard"

printf 'FCD backing path: %s\n' "$FCD_PATH"
```

# Create the helper VM in inventory accessible to the datastore containing the FCD and attach the existing disk without creating a linked delta:

```
govc vm.create \
  -on=false \
  "$HELPER_VM"

govc vm.disk.attach \
  -vm "$HELPER_VM" \
  -disk "$FCD_PATH" \
  -link=false

govc device.info -vm "$HELPER_VM" disk-*
```

# Verify that the helper VM is powered off:
```
test "$(govc vm.info -json "$HELPER_VM" | jq -r '.VirtualMachines[0].Runtime.PowerState')" = "poweredOff"
```

# Delete the source VKS PVC and PV
# The source workload must already be quiesced and the Velero backup completed.

# Delete the source VKS PVC and PV:

```
vks-src -n "$SRC_VKS_NS" delete pvc "$SRC_VKS_PVC" \
  --wait=true

if vks-src get pv "$SRC_VKS_PV" >/dev/null 2>&1; then
  vks-src delete pv "$SRC_VKS_PV" --wait=true
fi
```

# The Supervisor PVC may disappear automatically. If it remains, remove it using the normal namespaced Supervisor context:
```
if sup-src -n "$SRC_SUP_NS" get pvc "$SUP_PVC" >/dev/null 2>&1; then
  sup-src -n "$SRC_SUP_NS" delete pvc "$SUP_PVC" --wait=true
fi
```




```
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


