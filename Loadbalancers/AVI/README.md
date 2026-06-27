# Avi Load Balancer


## Download the Avi OVA
```
http://support.broadcom.com
Downloads
Search fro Avi
Select VMware Avi Load balancer
VMware Avi Load balancer
32.1.1
Terms and Conditions
I agree
VMware Controller OVA (you may have to make the screen real wide to see the download cloud)
```

## Install the Avi OVA
```
Burger Menu -> Inventory -> Compute -> vCenter Cluster Right click -> Deploy OVF Template
Local File
Select Avi OVA from dowloads folder
Virtual machine Name: avi
Cluster
vSAN Storage
management network: mgmt-vds01-wld01-01a
Enter Password twice (Famouse VMware password)
IP: 10.1.1.50
Mask: 255.255.255.0
GW.: 10.1.1.1.
Hostname of Avi Controller: avi
Finish
```
## Power up the Avi VM
```
Burger Menu -> Inventory -> Compute -> Select Avi VM right click power -> Power on
```

## Connect to Avi
```
Firefox Browser: http://10.1.1.50
admin/( VMware password)
vmware password x3
vmware password x3
Pass Phrase: vmware password x3
Confirm Phrase: vmware password x3
DNS: 10.1.1.1
DNS Search Domain: vcf.lab
Uncheck Join VMware...
Email: Next
Multi Tenant: Don't change
Select check box for set up cloud after
Save

Infrastructure -> Clouds
Create -> VMware
Name: mycloud
Object Name Prefix: mycloud
uncheck enable DHCP
uncheck IPv6 auto config
uncheck enable IPv6
check use static routes
Set Credentials
vc-wld01-a.site-a-vcf.lab
administrator@WLD.SSO
(Vmware password twice)
uncheck content lib
Save and Relaunch

Management network: mgmt-vds01-wld01-01a

Infrastructure -> Serives Engine Group -> myclud -> create 
Name: myservieenginegroup
Scope -> Host and Cluster -> select the vCetner cluster
Include the 3 hosts
Datastore shared -> Include -> vSAN Storage
Save

Infrastructure -> Cloud -> my cloud -> select the myservice engine group in SE templates -> SAVE
Infrastructure -> Network -> my cloud
Select edit on management network -> edit
add IP address pool edit
10.1.1.110-10.1.1.115 Use for SE vNIC
Save -> Save

Select the vSAN network -> edit
Uncheck use static IP for VIP and SE
Subnet: 10.1.4.0/25
Add pool: 10.1.4.40-10.1.4.49 Use for VIP
Save -> Save

Infrastructure -> Cloud -> my cloud -> edit -> IPAM / DNS
IPAM profile create
Mycloud
Name: myipam
Select the vSAN network
Save

Administration -> System Settings -> Edit
Access delete the 2 certs ssl / tls cert
3 dots -> edit
Name: mycert
Common name: avi
Add SAN: 10.1.1.50
Save -> Save
Log back in (session is diconnected due to new cert)
Allow basic auth check box
Save
```

# Enable Supervisor
```
Burger Menu -> Supervisor Management 

```

## Notes on networking 
```
#Sup:
#MGT 10.1.1.51-10.1.1.59	255.255.255.0	GW.:10.1.1.1 mgmt-vds01-wld01-01a
#Wrk 10.1.4.10-10.1.4.29	255.255.255.128	GW.: 10.1.4.1 vsan-vds02-wld01-01a


#MGT 10.1.1.110-10.1.1.125	255.255.255.0		GW.: 10.1.1.1 mgmt-vds01-wld01-01a
#Wkt 10.1.4.30-10.1.4.39		255.255.255.128		GW.: 10.1.4.1 vsan-vds02-wld01-01a
#vip 10.1.4.40-10.1.4.49		255.255.255.128		
#Tran 10.1.5.10-10.1.5.19	255.255.255.128		GW.: 10.1.5.1 vmotion-vds01-wld01-01a

```




