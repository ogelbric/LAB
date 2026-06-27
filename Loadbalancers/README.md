# Foundational Loadbalancer (One Arm)

The idea is to use a HOL / Field demo lab to reuse it for different Loadbalancer options to be tried out.

## As a Broadcom user find the lab(s) 
```
mylogin.broadcom.com
My applications search for demo
VCF Field Demo Lab
Select VCF-9.1 Single Site with Auto
```

## Once the lab is up use Firefox to bring up the workload vCenter
```
Delete the current Supervisor cluster
  Burger Menu -> Supervisor Manageent -> select supervisor cluster -> Deactivate
Enable DRS on vCenter cluster - Fully Automated
  Burger Menu -> Inventory -> Compute -> vcenter Cluster -> Configure -> vSphere DRS -> Edit -> Select Fully Automated
```

## Selecting Networks
```
Burger Menu -> Inventory -> Networks -> expand the 2 VDS networks -> select each one and verify the VLANS in the Distributed Port Group Details

mgmt-vds01-wld01-01a     VLAN 10		10.1.1.1 	255.255.255.0           .50-59 .110-.130		  MGT
vmotion-vds01-wld01-01a  VLAN 17		10.1.5.1	255.255.255.128							          Transit
vsan-vds02-wld01-01a     VLAN 15		10.1.4.1	255.255.255.128			.10-.99				      Workload/VIP
```

## Locating open IP ranges on the Router
```
SSH to router: ssh root@10.1.10.1 (Typical vmware password twice)

Investgating the routing table and availabe VLAN(s)

netstat -nr

Find open IP ranges: 

arp -a | grep 10.1.4
arp -a | grep 10.1.5
arp -a | grep 10.1.1
```


Enable supervisor cluster 
Cluster deployment 
Foundational load balancer

DNS 10.1.1.1
NTP 10.1.1.1
vcf.lab

Sup:
MGT 10.1.1.50-10.1.1.59	255.255.255.0	GW.:10.1.1.1
Wrk 10.1.4.10-10.1.4.29	255.255.255.128	GW.: 10.1.4.1

One Arm FLB:

MGT 10.1.1.110-10.1.1.125	255.255.255.0		GW.: 10.1.1.1
Wkt 10.1.4.30-10.1.4.39		255.255.255.128		GW.: 10.1.4.1
vip 10.1.4.40-10.1.4.49		255.255.255.128


Two Arm FLB:

MGT 10.1.1.110-10.1.1.125	255.255.255.0		GW.: 10.1.1.1
Wkt 10.1.4.30-10.1.4.39		255.255.255.128		GW.: 10.1.4.1
vip 10.1.4.40-10.1.4.49		255.255.255.128		
Tran 10.1.5.10-10.1.5.19	255.255.255.128		GW.: 10.1.5.1

Terminal: 	

vcf context create sup --endpoint 10.1.4.41 --insecure-skip-tls-verify --auth-type basic
#administrator@WLD.SSO/VMware123!VMware123!
vcf context use
k get nodes







