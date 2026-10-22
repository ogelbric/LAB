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
export SUP_CONTEXT="sup4"
export SRC_CONTEXT="cluster-a4"
export DST_CONTEXT="cluster-b4"
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

# kill the busybox POD

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

# Because the Supervisor PV still has reclaim policy Delete, the vSphere CSI provisioner will try to delete the CNS volume. The helper-VM attachment must prevent the backing FCD from being destroyed.

# Verify that the Supervisor PV remains in Released state and still references the expected volume ID:
```
sup-src get pv "$SUP_PV" \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,RECLAIM:.spec.persistentVolumeReclaimPolicy,HANDLE:.spec.csi.volumeHandle

test "$(sup-src get pv "$SUP_PV" -o jsonpath='{.status.phase}')" = "Released"
test "$(sup-src get pv "$SUP_PV" -o jsonpath='{.spec.csi.volumeHandle}')" = "$FCD_UUID"
```

# Inspect the PV events:
```
sup-src describe pv "$SUP_PV"
```
# The validated workflow showed repeated VolumeFailedDelete events containing:

```
ResourceInUse
The resource 'volume' is in use.
Do not continue if the backing disk has disappeared or if the source PV is not in the expected guarded state.
```

# Remove the CNS registration but keep the FCD
# This is the handover operation:
```
govc volume.rm -keep "$FCD_UUID"
# -keep is essential. It instructs CNS to remove the container-volume registration without deleting the backing disk.
```

# Verify that the CNS volume registration is gone:
```
if govc volume.ls "$FCD_UUID" >/dev/null 2>&1; then
  echo "ERROR: CNS volume is still registered" >&2
  exit 1
fi
```
# Then verify independently that the backing FCD still exists:
```
govc disk.ls -l "$FCD_BACKING_ID"
govc device.info -vm "$HELPER_VM" disk-*
```
# At this point the desired state is:
```
old CNS registration: gone
backing FCD:           present
helper VM:             powered off, FCD still attached
```

# Wait for the old Supervisor PV to disappear
# Once CNS no longer contains the old container-volume registration, the source CSI deletion can finish reconciling the stale Supervisor PV without destroying the retained FCD.

# Wait for the old Supervisor PV to be removed:
```
sup-src wait \
  --for=delete \
  "pv/${SUP_PV}" \
  --timeout=5m
```
# Confirm that the old PV no longer blocks registration of the same volume ID:
```
if sup-src get pv "$SUP_PV" >/dev/null 2>&1; then
  echo "ERROR: old Supervisor PV still exists" >&2
  exit 1
fi
```
# This is the critical difference from the old Retain workflow: there is no retained Supervisor PV and therefore no cluster-scoped claimRef to edit.

# Detach the retained FCD from the helper VM
# Only detach the helper disk after the old CNS registration and old Supervisor PV are gone.

# Identify the helper VM disk device and remove it while keeping the backing files:
```
govc device.info -vm "$HELPER_VM" disk-*

export HELPER_DISK_DEVICE=$(
  govc device.ls -vm "$HELPER_VM" | awk '/^disk-/ {print $1; exit}'
)

test -n "$HELPER_DISK_DEVICE" || {
  echo "ERROR: helper VM disk device not found" >&2
  exit 1
}

govc device.remove \
  -vm "$HELPER_VM" \
  -keep \
  "$HELPER_DISK_DEVICE"
```

# Verify that the retained FCD still exists:
```
govc disk.ls -l "$FCD_BACKING_ID"
```
# The helper VM can be removed after the migration has been validated.

# Register the retained FCD in the destination Supervisor namespace
# Create a namespaced CnsRegisterVolume object:
```
sup-dst -n "$DST_SUP_NS" apply -f - <<EOF_REGISTER
apiVersion: cns.vmware.com/v1alpha1
kind: CnsRegisterVolume
metadata:
  name: ${REGISTER_NAME}
spec:
  volumeID: "${FCD_UUID}"
  accessMode: ${SUP_ACCESS_MODE}
  pvcName: ${DST_SUP_PVC}
EOF_REGISTER
```
# Wait for registration:
```
sup-dst -n "$DST_SUP_NS" wait \
  --for=jsonpath='{.status.registered}'=true \
  "cnsregistervolume/${REGISTER_NAME}" \
  --timeout=10m
```
# Inspect the result and fail if the controller reports an error:
```
sup-dst -n "$DST_SUP_NS" get \
  "cnsregistervolume/${REGISTER_NAME}" \
  -o yaml

REGISTER_ERROR=$(
  sup-dst -n "$DST_SUP_NS" get \
    "cnsregistervolume/${REGISTER_NAME}" \
    -o jsonpath='{.status.error}'
)

test -z "$REGISTER_ERROR" || {
  echo "ERROR: CnsRegisterVolume failed: $REGISTER_ERROR" >&2
  exit 1
}
```
# Discover the newly created Supervisor PV:
```
export DST_SUP_PV=$(
  sup-dst -n "$DST_SUP_NS" get pvc "$DST_SUP_PVC" \
    -o jsonpath='{.spec.volumeName}'
)

sup-dst -n "$DST_SUP_NS" get pvc "$DST_SUP_PVC" \
  -o custom-columns=NAME:.metadata.name,PV:.spec.volumeName,PHASE:.status.phase

sup-dst get pv "$DST_SUP_PV" \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,HANDLE:.spec.csi.volumeHandle
Confirm that the destination Supervisor PVC is Bound and that the new Supervisor PV represents the retained FCD:

test "$(
  sup-dst -n "$DST_SUP_NS" get pvc "$DST_SUP_PVC" \
    -o jsonpath='{.status.phase}'
)" = "Bound"

test "$(
  sup-dst get pv "$DST_SUP_PV" \
    -o jsonpath='{.spec.csi.volumeHandle}'
)" = "$FCD_UUID"
```

# The VKS volume handle is now the new Supervisor PVC name:
```
export DEST_VOLUME_HANDLE="$DST_SUP_PVC"
```

# Create the destination VKS PV and PVC
# For this workflow, the VKS PV volumeHandle is the newly registered destination Supervisor PVC name ${DST_SUP_PVC} in ${DST_SUP_NS}. 
# Do not put the FCD UUID here: that UUID is used by 
# CnsRegisterVolume and the Supervisor PV. Continue only after Step 12 confirms successful registration and a Bound destination Supervisor PVC.

```
vks-dst apply -f - <<EOF_DEST_VOLUME
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ${DST_VKS_PV}
spec:
  capacity:
    storage: ${VKS_PVC_SIZE}
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  claimRef:
    namespace: ${DST_VKS_NS}
    name: ${DST_VKS_PVC}
  csi:
    driver: csi.vsphere.vmware.com
    fsType: ext4
    volumeHandle: "${DEST_VOLUME_HANDLE}"
    volumeAttributes:
      type: "vSphere CNS Block Volume"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${DST_VKS_PVC}
  namespace: ${DST_VKS_NS}
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  volumeName: ${DST_VKS_PV}
  resources:
    requests:
      storage: ${VKS_PVC_SIZE}
EOF_DEST_VOLUME

vks-dst -n "$DST_VKS_NS" wait \
  --for=jsonpath='{.status.phase}'=Bound \
  "pvc/${DST_VKS_PVC}" \
  --timeout=5m 
```

# Restore application manifests
# Restore application metadata only after the destination VKS PV/PVC has been created.
```
velero restore create "$RESTORE_NAME" \
  --kubeconfig "$DST_VKS_KUBECONFIG" \
  --from-backup "$BACKUP_NAME" \
  --namespace-mappings "${SRC_VKS_NS}:${DST_VKS_NS}" \
  --exclude-resources \
persistentvolumes,persistentvolumeclaims,volumesnapshots.snapshot.storage.k8s.io,volumesnapshotcontents.snapshot.storage.k8s.io \
  --wait

velero restore describe "$RESTORE_NAME" \
  --kubeconfig "$DST_VKS_KUBECONFIG" \
  --details
```

# Mount the migrated volume and verify the data
```
vks-dst -n "$DST_VKS_NS" apply -f - <<EOF_READER
apiVersion: v1
kind: Pod
metadata:
  name: migration-test-reader
spec:
  restartPolicy: Never
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c"]
      args: ["cat /data/migration-test.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ${DST_VKS_PVC}
EOF_READER

vks-dst -n "$DST_VKS_NS" wait \
  --for=condition=Ready \
  pod/migration-test-reader \
  --timeout=5m

vks-dst -n "$DST_VKS_NS" exec migration-test-reader -- \
  cat /data/migration-test.txt

```  

# Cleanup after validation
# Delete the test reader after validation:
```
vks-dst -n "$DST_VKS_NS" delete pod migration-test-reader
```

# There is no migration snapshot or retained source Supervisor PV to remove. After the migrated workload has been validated, remove the powered-off helper VM if it is no longer required:
```
govc vm.destroy "$HELPER_VM"
```
# Only destroy the helper VM after its migration-guard disk has been detached with device.remove -keep and the destination workload has been validated.

# Validation checklist
```
 Source workload was quiesced and manifest-only Velero backup completed.
 VKS PV, Supervisor PVC, Supervisor PV, CNS volume ID and FCD backing object were recorded.
 Powered-off helper VM held the FCD before source storage teardown.
 Source Supervisor PV entered Released and CSI deletion failed safely with ResourceInUse.
 govc volume.rm -keep removed the old CNS registration while the FCD remained present.
 Old Supervisor PV disappeared before the helper VM released the disk.
 CnsRegisterVolume created a new Supervisor PVC/PV in the destination namespace.
 New Supervisor PV references the original FCD UUID.
 Destination VKS PV uses the new Supervisor PVC name as volumeHandle.
 Destination workload mounted and validated the original data.
```



```
```








# For later to figgure out on how to do the login  with VCF commands
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


