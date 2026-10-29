# VCF CLI with cert

## 3 Step process Application onto Kubernetes cluster

### 1 Supervisor Cluster
```
#
# API end point is supervisor IP cert 
#
openssl s_client -connect 10.1.8.132:443 -showcerts </dev/null 2>/dev/null  | openssl x509 -outform PEM > supervisor.crt
#
# Connecting to supervisor cluster
#
vcf context create  mysupervisor01 --endpoint https://10.1.8.132 --username administrator@WLD.SSO --ca-certificate supervisor.crt --auth-type basic
#
vcf context list
vcf context use
kubectl get nodes
#
vcf clusters list -A
```

### 2 Guest Cluster

```
#
# Connecting to guest cluster
#
vcf context create  myguestcluster01 --endpoint https://10.1.8.132 --username administrator@WLD.SSO --ca-certificate supervisor.crt --auth-type basic --workload-cluster-name orfk8 --workload-cluster-namespace orfns
vcf context list
vcf context use myguestcluster01:orfk8
kubectl get nodes 
```

### 3 Apply Application

```
kubectl apply -f {application file yaml}

https://github.com/ogelbric/LAB/tree/main/where_is(are)/SampleApp

```


