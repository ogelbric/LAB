
# Installing VCF CLI in VMware Lab Platform

## Creating a VMservice OS image
```
Download an image and import into content lib in vCenter and attach the content lib for the VMservice Tile to the vSphere namespace

https://github.com/vmware/photon/wiki/Downloading-Photon-OS

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






