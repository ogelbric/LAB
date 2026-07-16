# Adding AVI to a full NSX stack build out. 

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

### AVI License
```
Administration -> Licensing -> OnPrem Licenses HUB 
(Provides 20 SU's) 
```

