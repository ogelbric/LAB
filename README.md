# LAB

Create dev namespace!
```
vcf context create vcfa --endpoint auto-a.site-a.vcf.lab --api-token Gzy8gHueTtYu20ODUJfx3asdZ7NWveuC --tenant-name broadcom --ca-certificate vcfa-cert-chain.pem
```
```
vcf context use
vcf cluster list
```
Create vks-01 cluser!
```
vcf cluster register-vcfa-jwt-authenticator vks-01
vcf cluster kubeconfig get vks-01 --export-file ~/.kube/config
cat ~/.kube/config | grep vks-01 | grep @
vcf context create vks-01 --kubeconfig ~/.kube/config --kubecontext vcf-cli-vks-01-dev-f5llm-vks-01-dev-f5llm
vcf context refresh
vcf context use vks-01
kubectl get nodes
```



