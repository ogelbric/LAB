# LAB

```
vcf context create vcfa --endpoint auto-a.site-a.vcf.lab --api-token Gzy8gHueTtYu20ODUJfx3asdZ7NWveuC --tenant-name broadcom --ca-certificate vcfa-cert-chain.pem
```
```
vcf context use
```
```
vcf cluster list
```
```
vcf cluster register-vcfa-jwt-authenticator kubernetes-cluster-bkec
```
```
vcf cluster kubeconfig get kubernetes-cluster-bkec --export-file ~/.kube/config
```
```
cat ~/.kube/config | grep kuber | grep @
```
```
vcf context create vks-01 --kubeconfig ~/.kube/config --kubecontext vcf-cli-kubernetes-cluster-bkec-dev-f5llm@kubernetes-cluster-bkec-dev-f5llm
```
```
vcf context refresh
```



