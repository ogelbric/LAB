# LAB

Create dev namespace!
```
vcf context create vcfa --endpoint auto-a.site-a.vcf.lab --api-token Gzy8gHueTtYu20ODUJfx3asdZ7NWveuC --tenant-name broadcom --ca-certificate vcfa-cert-chain.pem
```
```
vcf context use
vcf cluster list
```
Create a Namespace (dev) in VCF-A and then a K8 cluster (vks-01)!
```
vcf cluster register-vcfa-jwt-authenticator vks-01
vcf cluster kubeconfig get vks-01 --export-file ~/.kube/config
cat ~/.kube/config | grep vks-01 | grep @
vcf context create vks-01 --kubeconfig ~/.kube/config --kubecontext vcf-cli-vks-01-dev-f5llm-vks-01-dev-f5llm
vcf context refresh
vcf context use vks-01
kubectl get nodes
```
Install standard packages
```
vcf package repository add default-repo --url projects.packages.broadcom.com/vsphere/supervisor/vks-standard-packages/3.6.0-20260211/vks-standard-packages:3.6.0-20260211 -n tkg-system
vcf package available list -n tkg-system
cd Documents/Lab
kubectl create ns prometheus-installed
vcf package install prometheus -p prometheus.kubernetes.vmware.com --values-file prometheus-data-values.yaml -n prometheus-installed -v 3.5.0+vmware.1-vks.2
kubectl get pods -n tanzu-system-monitoring
```
```
kubectl create ns telegraf-installed
vcf package install telegraf -p telegraf.kubernetes.vmware.com --values-file telegraf-data-values.yaml -n telegraf-installed -v 1.37.1+vmware.1-vks.1
kubectl get pods -n tanzu-system-telegraf
```
Opencart
```
kubectl create namespace opencart
kubectl label ns opencart pod-security.kubernetes.io/enforce=privileged
kubectl apply -f opencart-lb.yaml -n opencart
kubectl get service -n opencart

```

Supervisor cluster
```
vcf context create sup --endpoint 10.1.0.6 --insecure-skip-tls-verify --auth-type basic
vcf context use sup
```

Simple app
```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: http-echo
  name: http-echo
spec:
  ports:
    - port: 80
  selector:
    app: http-echo
  type: LoadBalancer

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-echo
spec:
  selector:
    matchLabels:
      app: http-echo
  replicas: 3
  template:
    metadata:
      labels:
        app: http-echo
    spec:
      containers:
      - name: ttools
        #image: harbor.lab.local/vks/hello-app
        image: gcr.io/boreal-rain-256712/http-echo
        ports:
        - containerPort: 80
```

