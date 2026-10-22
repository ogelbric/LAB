

# Set up all the contexts

```
#
#save off the old .kube/config file
#
mv /${HOME}/.kube/config /${HOME}/.kube/config.bak
rm /${HOME}/.kube/config
#
export SUP_CONTEXT="sup2"
export SRC_CONTEXT="cluster-a2"
export DST_CONTEXT="cluster-b2"
#
export SRC_CLUSTER="cluster-a"
export DST_CLUSTER="cluster-b"
export SRC_VSPHERE_NS="namespace-a"
export DST_VSPHERE_NS="namespace-b"
#
vcf context create ${SUP_CONTEXT} --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic
vcf context use ${SUP_CONTEXT}
kubectl get nodes
cp /${HOME}/.kube/config "$HOME/supervisor-kubeconfig"
export SUPERVISOR_KUBECONFIG="$HOME/supervisor-kubeconfig"
rm /${HOME}/.kube/config
#
vcf context create ${SRC_CONTEXT} --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name ${SRC_CLUSTER} --workload-cluster-namespace ${SRC_VSPHERE_NS}
cp /${HOME}/.kube/config "$HOME/source-vks-kubeconfig"
export SRC_VKS_KUBECONFIG="$HOME/source-vks-kubeconfig"
vcf context use ${SRC_CONTEXT}:${SRC_CLUSTER}
kubectl get nodes
rm /${HOME}/.kube/config
#
vcf context create ${DST_CONTEXT} --endpoint https://10.1.8.132 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name ${DST_CLUSTER} --workload-cluster-namespace ${DST_VSPHERE_NS}
cp /${HOME}/.kube/config "$HOME/destination-vks-kubeconfig"
export DST_VKS_KUBECONFIG="$HOME/destination-vks-kubeconfig"
vcf context use ${DST_CONTEXT}:${DST_CLUSTER}
kubectl get nodes
rm /${HOME}/.kube/config



```

