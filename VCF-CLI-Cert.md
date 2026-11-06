# VCF CLI with cert

## 3 - Step process Application onto Kubernetes cluster

### 1 - Supervisor Cluster / vSphere Namespace
```
export API_IP="10.1.8.132"
export USER='administrator@WLD.SSO'
export NS="orfns"
export CL="orfk8"
export SUP="mysupervisor01"
export GC="myguestcluster01"

#
# API end point is supervisor IP cert 
#
openssl s_client -connect ${API_IP}:443 -showcerts </dev/null 2>/dev/null  | openssl x509 -outform PEM > supervisor.crt
#
# Connecting to supervisor cluster
#
vcf context create  ${SUP} --endpoint https://${API_IP} --username ${USER} --ca-certificate supervisor.crt --auth-type basic
#
vcf context list
vcf context use      # use the context of the supervisor:vSphereNameSpace
vcf context use ${SUP}:${NS}
kubectl get nodes
#
vcf clusters list -A
#
# build new cluster (make sure the Harbor cert is in the Supervisor then Harbor cert will end up in the guest cluster!)
#
kubectl apply -f {Sample Cluster yaml}
#https://github.com/ogelbric/LAB/tree/main/where_is(are)/SampleClusteryaml

vcf clusters list -A
```

### 2 - Guest Cluster

```
#
# Connecting to guest cluster
#
vcf context create  ${GC} --endpoint https://${API_IP} --username ${USER} --ca-certificate supervisor.crt --auth-type basic --workload-cluster-name ${CL} --workload-cluster-namespace ${NS}
vcf context list
vcf context use ${GC}:${CL}
kubectl get nodes 
```

### 3 - Apply Application

```
kubectl apply -f {application file yaml}

https://github.com/ogelbric/LAB/tree/main/where_is(are)/SampleApp

```

### Apendix - Harbor Supervisor Service 

```
https://github.com/ogelbric/LAB/tree/main/where_is(are)/Create_Harbor
```

### Apendix - Build test image

```
https://github.com/AndriyKalashnykov/vks-guides/blob/main/docs/IMAGE-DOCKER.md
```

### Apendix - Get Guest Cluster kube config file

```
export API_IP="10.1.8.132"
export USER='administrator@WLD.SSO'
export NS="orfns"
export CL="orfk8"
export SUP="mysupervisor04"
export GC="myguestcluster01"

#
# API end point is supervisor IP cert 
#
openssl s_client -connect ${API_IP}:443 -showcerts </dev/null 2>/dev/null  | openssl x509 -outform PEM > supervisor.crt
#
# Connecting to supervisor cluster
#
vcf context create  ${SUP} --endpoint https://${API_IP} --username ${USER} --ca-certificate supervisor.crt --auth-type basic
#
vcf context use ${SUP}:${NS}
#
kubectl get secret ${CL}-kubeconfig -n ${NS} -o jsonpath='{.data.value}' | base64 --decode > /tmp/${NS}-${CL}-direct.kubeconfig
#
kubectl --kubeconfig="/tmp/${NS}-${CL}-direct.kubeconfig" get pods -A
#
kubectl --kubeconfig="/tmp/${NS}-${CL}-direct.kubeconfig" get svc -A



```


