# Adding AVI to a full NSX stack build out. 

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

## NSX (namgement) 
System -> Fabric -> Trasport zones -> nsx-vlan-trasport zone (for AVI SE's) 
System -> Fabric -> Hosts -> Transport node profile -> edit
Seelct (1)  on host switch -> edit 
Add to vlan transport zone nsx-vlan-trasport zone
Add -> apply -> save 


```








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






