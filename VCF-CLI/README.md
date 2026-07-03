
# Installing VCF CLI in VMware Lab Platform

The idea is to have a virgin Linux VM to install the VCF cli and the VCF plugins in an airgapped environment. 


## Creating a VMservice OS image
```
Create namespace1000 in vCenter 

Download the Photon OS image and import into content lib in vCenter and attach the content lib for the VMservice Tile to the vSphere namespace1000

https://github.com/vmware/photon/wiki/Downloading-Photon-OS
Get: OVA with virtual hardware v15 (Supports both BIOS and UEFI boot, default is UEFI)

Deploy a VM from the VMservce from the vCenter Consumption Interface (yaml(s) for VM are in this GitHub folder) into namespace1000 (create namespace1000)
Obtain the LB IP for VM from namespace 
ssh orf@10.1.4.11
```

## Download the VCF CLI from Broadcom
```
https://support.broadcom.com
Serach for: VMware Cloud Foundation
9.1.00 > VCF Consumption CLI & VCF CLI Plugins
```

## Move VCF CLI and Plugins to new jump box
```
scp VCF-* orf@10.1.4.11:/home/orf/.

VCF-Consumption-CLI-Linux_AMD64-9.1.0.0.25296 100%   44MB  72.0MB/s   00:00    
VCF-Consumption-CLI-PluginBundle-Linux_AMD64- 100%  366MB 134.1MB/s   00:02
```

## Install a few things on the Photon OS VM
```
ssh root@10.1.4.11
changeme
new password x2

tdnf install sudo
tdnf install tar
tdnf install wget
curl -L https://istio.io/downloadIstio | sh

ssh orf@10.1.4.11
tar -xvf VCF-Consumption-CLI-Linux_AMD64-9.1.0.0.25296329.tar.gz
sudo install vcf-cli-linux_amd64 /usr/local/bin/vcf
vcf version

#version: v9.1.0.0.25296329
#buildDate: 2026-03-20
#...
```

## Plugins
```
rm -rf ~/.local/vcf
rm -rf ~/.local/vcf-cli-telemetry

mkdir -p VCF-Consumption-CLI-Plugins-9.1
tar -zxvf VCF-Consumption-CLI-PluginBundle-Linux_AMD64-9.1.0.0300.25509668.tar.gz -C VCF-Consumption-CLI-Plugins-9.1
vcf plugin install all --local-source VCF-Consumption-CLI-Plugins-9.1
vcf plugin list
```


## Log into sup cluster
```
vcf context create  --endpoint https://10.1.4.41 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic

Provide a name for the context:  sup66
vcf context list
vcf context use
#or
vcf context use sup66:namespace1000
```

## Install the kubectl command from API endpoint
```
wget --no-check-certificate https://10.1.4.41/wcp/plugin/linux-amd64/vsphere-plugin.zip
unzip vsphere-plugin.zip
cd bin
sudo mv kubectl kubectl-vsphere /usr/local/bin/
sudo chmod +x /usr/local/bin/kubectl /usr/local/bin/kubectl-vsphere
```
## Use kubectl
```
kubectl get nodes
```

```
NAME                               STATUS   ROLES                  AGE   VERSION
42275129b28d6a90e230a1623ca6c164   Ready    control-plane,master   32h   v1.32.9+vmware.2-fips
esx-05a.site-a.vcf.lab             Ready    agent                  32h   v1.32.5-sph-f4e887d
esx-06a.site-a.vcf.lab             Ready    agent                  31h   v1.32.5-sph-f4e887d
esx-07a.site-a.vcf.lab             Ready    agent                  31h   v1.32.5-sph-f4e887d
```

## Loginto a Guest cluster
```
In vCenter in the Consumption interface create a guest cluster in namespace1000
alias k=kubectl

vcf cluster list

#
# In there there may be a token refresh necessary in order not to get a pinnepad error
# Need command here (vcf context refresh # seem not to do the trick
#

vcf context create  --endpoint https://10.1.4.41 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name kubernetes-cluster-5smg --workload-cluster-namespace namespace1000

guest44


k get nodes
# Should disply the supervisor control and worker nodes (ESX)

vcf context list

#  NAME                             CURRENT  TYPE        
#  guest44                          false    kubernetes  
#  guest44:kubernetes-cluster-5smg  true     kubernetes  
#  sup66                            false    kubernetes  
#  sup66:namespace1000              false    kubernetes  
#  sup66:svc-cci-ns-2q0cm           false    kubernetes  
#  sup66:svc-tkg-1g5q6              false    kubernetes  
#  sup66:svc-velero-xfbxl           false    kubernetes  

vcf context use
# Select the guest44 context

k get nodes
# Should get you the gues cluster controll plane nodes and worker nodes

k get pods -A
```

# Install Service Mesch

## Scale out guest cluster
```
# Swap to supervisor cluster
vcf context use sup66:namespace1000
# add 2 more workers for a total of 3
vcf cluster scale kubernetes-cluster-5smg -w 3 -n namespace1000
vcf context use guest44:kubernetes-cluster-5smg
k get nodes 
```

## Look at packages 
```
#Adding the external packe repo (for airgapped this would have to be moved to an internal habor repo
vcf package repository add vks-packages --url projects.packages.broadcom.com/vsphere/supervisor/vks-standard-packages/3.6.0-20260320/vks-standard-packages:3.6.0-20260320 -n tkg-system

kubectl get pkgr -n tkg-system

# Make sure the repo is working
kubectl describe pkgr vks-packages -n tkg-system|grep image

#find the istio package
vcf package available list -n tkg-system |grep istio

#List versions and we need 1.28
vcf package available get istio.kubernetes.vmware.com -n tkg-system

# Get the data vaues file
vcf package available get istio.kubernetes.vmware.com/1.28.2+vmware.1-vks.1 --default-values-file-output istio-data-values.yaml -n tkg-system
```

# Here is Bob(s) short version
```
 istio:
   enableStrictMTLS: true
   gateways:
     egress:
       autoscaling:
         enabled: false
         maxReplicas: 5
         minReplicas: 1
       enabled: true
       replicas: 1
#       resources:
#         limits:
#           cpu: 2000m
#           memory: 1024Mi
#         requests:
#           cpu: 100m
#           memory: 128Mi
     ingress:
       autoscaling:
         enabled: false
         maxReplicas: 5
         minReplicas: 1
       enabled: false
       replicas: 1
#       resources:
#         limits:
#           cpu: 2000m
#           memory: 1024Mi
#         requests:
#           cpu: 100m
#           memory: 128Mi
   meshConfig:
     accessLogFile: /dev/stdout
     connectTimeout: 10s
     enableDNSProxy: false
     enablePrometheusMerge: true
     enableTracing: true
     externalIstiod: false
     meshMTLS:
       minProtocolVersion: TLSV1_2
     trustDomain: cluster.local
   namespace: istio-system
```

## Deploy istio
```
vcf package install istio \
-p istio.kubernetes.vmware.com \
-v 1.28.2+vmware.1-vks.1 \
--values-file bobsistio.yaml \
-n tkg-system

# You should see a deploy succeeded
```

## Make sure istio is running 
```
kubectl get pkgi -n tkg-system |grep istio
kubectl get po -n istio-system
kubectl get po -n istio-egress
```

## Deploy sample app 
```
# Enable istio injection
kubectl label namespace default istio-injection=enabled
# Deploy sample app
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/refs/heads/main/release/kubernetes-manifests.yaml
```

## Check out the app 
```
kubectl get svc; echo "---"; kubectl get svc | grep front
```

## Create Gateway
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: istio-ingress
  labels:
    # Required in VKS to allow Istio proxies to run
    pod-security.kubernetes.io/enforce: privileged
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: istio-ingress
spec:
  gatewayClassName: istio 
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    # Shared model accepts any subdomain under vdoubleb.com
    hostname: "*.vksc01.examle.com" 
    allowedRoutes:
      namespaces:
        from: All
EOF
```

## Check the Gateway Infrastructure
```
kubectl get pods,gateway -n istio-ingress
```

# Update Local dns with *.vksc01.example.com 
In this lab the DNS is not accessible so next thing is update /etc/hosts with shop.vksc01.example.com
```
kubectl get pods,gateway -n istio-ingress | tail -1 | awk '{print $3}'
# 10.1.4.45

echo `kubectl get pods,gateway -n istio-ingress | tail -1 | awk '{print $3}'` "shop.vksc01.example.com" | sudo tee -a /etc/hosts
```

## Test ingress
Since DNS is not available the test is only partially working and gets a 404 but does connect 
```
curl -vI shop.vksc01.example.com
```







