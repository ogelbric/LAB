# Adding AVI to a full NSX stack build out. (Using the "VCF Experience Program: Kubernetes (9.1)"image in HOL lab)

## vCenter 
### External Connection
```
vSphere management cluster (not vSphere workload cluster)
vCenter server in inventory -> configure -> networking -> external connection
add an external connection mynewconnection / vlan 11 / 10.1.2.1/24
```
### VNA Cluster
```
vSphere management cluster (not vSphere workload cluster)
vCenter server in inventory -> configure -> networking -> VNA Cluster -> add cluster
Give it a name: vna1.vcf.lab / DHCP / mgt network / storage -> save (you can have just one)
In my case I got 10.1.1.220 
```
### Create Content lib for AVI
```
Burger Menu -> content lib -> name: myaviSElib -> use all defaults -> create
```
### Create VKR content lib for supervisor cluster
```
Burger Menu -> content lib -> name: supcluster -> subscribe: wp-content.vmware.com/v2/latest/lib.json  -> create
```

## NSX (Management) 
```
System -> Fabric -> Trasport zones -> nsx-vlan-trasport zone (for AVI SE's) 
System -> Fabric -> Hosts -> Transport node profile -> edit
Seelct (1)  on host switch -> edit 
Add to vlan transport zone nsx-vlan-trasport zone
Add -> apply -> save 
```

## vSphere
```
Inventory -> vCenter -> Configure -> Networking -> VNA clusters -> make sure VNLA it is up and green 
Networking -> Virtual Private Clouds -> Configure -> IP Blocks -> Add IP block
Name: myexternalipblock
Visibility: External
Range: 10.1.3.40-10.1.3.50
Save

Networking -> Virtual Private Clouds -> Configure -> Connectivity Profiles -> Edit default profile 
Select Virtual Network Appliance Cluster: myvnacluster
My external IP block
N-S Services: on
Outbound NAT: on
My external IP block 

Networking -> Virtual Private Clouds -> Configure -> Service profile
Edit -> add NTP (10.1.1.1) and DNS (10.1.1.1) 
```
# VCF Ops
```
Build -> Lifecycle -> fleet instance -> Bianry management -> Install Images -> Filter on AVI -> 32.1.1
Management Domain -> manage components -> AVI Install -> Wizzard -> small -> Passwords -> Node IP (x3)
Register in DNS the avi.vcf.lab

```



# section below needs work 


## NSX Manager
```
System -> Apliances -> AVI Load Balancer
Assign Virtual IP: 10.1.2.40
Deploy AVI: (OVA has the downloaded from support.broadcom.com (serach for AVI))
IP: 10.1.2.41/25
GW: 10.1.2.1
Network: vsan
```

## AVI Appliance
```
Go to https://10.1.2.40 
And finish the set basic setup of AVI 
```

## AVI License
```
Administration -> Licensing -> OnPrem Licenses HUB 
(Provides 20 SU's) 
```

## AVI Connect to NSX

```
Infrastructure -> Get Started -> NSX Get Started quick Start Wizard -> Get Started
Cloud Name: mycloud
Prefix: mycloudnsx
Enter NSX host name: nsx-wld01-a.site-a.vcf.lab

```






