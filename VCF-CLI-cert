# VCF CLI with cert

```
#
# API end point is supervisor IP cert 
#
openssl s_client -connect 10.1.8.132:443 -showcerts </dev/null 2>/dev/null  | openssl x509 -outform PEM > supervisor.crt
#
# Connecting to supervisor cluster
#
vcf context create  supervisor01 --endpoint https://10.1.8.132 --username administrator@WLD.SSO --ca-certificate supervisor.crt --auth-type basic
#
vcf context list
vcf context use
kubectl get nodes
#
vcf clusters list -A 
#
# Connecting to guest cluster
#
vcf context create  guestcluster01 --endpoint https://10.1.8.132 --username administrator@WLD.SSO --ca-certificate supervisor.crt --auth-type basic --workload-cluster-name orfk8 --workload-cluster-namespace orfns
vcf context list
vcf context use guestcluster01:orfk8
kubectl get nodes 
```
