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
```


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






