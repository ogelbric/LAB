
# Installing VCF CLI
## Creating a VMservice OS image
```
Download an image and import into content lib and attach the content lib for the VM to the namespace

https://github.com/vmware/photon/wiki/Downloading-Photon-OS

Deploy a VM from the VMservce from the vCenter Consumption Interface
Obtain the LB IP for VM from namespace 
ssh orf@10.1.4.11

```

## Download the VCF CLI 
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

## Install a few things on this Photon OS
ssh root@10.1.4.11
changeme
new password x2

tdnf install sudo
tdnf install tar

ssh orf@10.1.4.11
tar -xvf VCF-Consumption-CLI-Linux_AMD64-9.1.0.0.25296329.tar.gz
sudo install vcf-cli-linux_amd64 /usr/local/bin/vcf
vcf version
```
```
version: v9.1.0.0.25296329
buildDate: 2026-03-20
...
```

## Plugins
```
mkdir plug
mv VCF-Consumption-CLI-PluginBundle-Linux_AMD64-9.1.0.0300.25509668.tar.gz ./plug/.
cd plug
tar xvf VCF-Consumption-CLI-PluginBundle-Linux_AMD64-9.1.0.0300.25509668.tar.gz 

sudo vcf plugin install all --local-source .



```






```




