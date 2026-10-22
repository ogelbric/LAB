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


