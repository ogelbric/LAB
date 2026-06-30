# Avi Load Balancer

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
Enable DRS on vCenter cluster - Fully Automated -> ok
  Burger Menu -> Inventory -> Compute -> vcenter Cluster -> Configure -> vSphere DRS -> Edit -> Select Fully Automated
```

## Selecting Networks
```
Burger Menu -> Inventory -> Networks -> expand the 2 VDS networks -> select each one and verify the VLANS in the Distributed Port Group Details

mgmt-vds01-wld01-01a     VLAN 10		10.1.1.1 	255.255.255.0           .50-59 .110-.130		  MGT
vmotion-vds01-wld01-01a  VLAN 17		10.1.5.1	255.255.255.128							          VIP
vsan-vds02-wld01-01a     VLAN 15		10.1.4.1	255.255.255.128			.10-.99				      Workload
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
DataCenter
Cluster
vSAN Storage
network tab
management network: mgmt-vds01-wld01-01a
Enter Password twice (Famouse VMware password)
IP: 10.1.1.50
Mask: 255.255.255.0
GW.: 10.1.1.1
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
Select check box for set up cloud after (next to save button)
Save

Administration -> Licensing -> On Prem Licensing Hup should give you 20 lic's

Infrastructure -> Clouds
Create -> VMware
Name: mycloud
Object Name Prefix: mycloud
uncheck enable DHCP
uncheck IPv6 auto config
uncheck enable IPv6
check use static routes
Set Credentials
vc-wld01-a.site-a.vcf.lab
administrator@WLD.SSO
(Vmware password twice)
uncheck content lib
Save and Relaunch
Cancel and yes to cont...

Infrastructure -> cloud Resources -> Serives Engine Group -> mycloud -> Default Group -> edit 
Scope -> Host and Cluster -> include -> select the vCenter cluster
Include the 3 hosts
Datastore shared -> Include -> vSAN Storage
Save

Infrastructure -> Cloud -> my cloud -> select the myservice engine group in SE templates
Management Network select the mgmt-vds01-wld01-01a
Save

Infrastructure -> Network -> my cloud
Select edit on management network -> edit
add IP address pool edit
Check box use for VIP and SE
add net
10.1.1.110-10.1.1.115
Save -> Save

Select the vmotion-vds01-wld01-01a -> edit
Pre-fix: 10.1.5.0/25
Add
Use static IP for VIP and SE
Add pool: 10.1.5.40-10.1.5.49 Use for VIP
Save -> Save

Infrastructure -> Cloud -> my cloud -> edit -> IPAM / DNS
IPAM profile create (3 buttons)
Name: myipam
Seelct Cloud: mycloud
Usable network -> add :  vmotion-vds01-wld01-01a
Save

Administration -> System Settings -> Edit
Access delete the 2 certs ssl / tls cert
3 dots -> edit -> Create
Name: mycert
Common name: avi
Add SAN: 10.1.1.50
Save -> Save
Log back in (session is diconnected due to new cert)
Allow basic auth check box ( *** Looks like this is not needed anymore *** )
Save

Create route from workload network to Frontend network
Infrastructure -> VRF Context -> mycloud -> edit global -> add static route
10.1.4.0/25 -> 10.1.5.0
```

# Enable Supervisor
```
Burger Menu -> Supervisor Management -> Get Started -> vSphere Distributed Switch -> Avi
Cluster Deployment
Name of Supervisor cluster: sup1
Zone vCenter Cluster
Storage 3x vSAN default
Name: avi
10.1.1.50:443
admin
(VMware password 3 times)

In AVI get the cert
  Templates -> security -> SSL/TLS/ Certs -> mycert -> export -> select the certificate -> copy to clip board

Paste into server certificate in vCetner and make sure the trailing CR is deleted !!!!!!!
Cloud Name: mycloud

Management Network
Static
#MGT 10.1.1.51-10.1.1.59	255.255.255.0	GW.:10.1.1.1 mgmt-vds01-wld01-01a

vcf.lab

Workload network
Static
work1
#Wrk 10.1.4.10-10.1.4.29	255.255.255.128	GW.: 10.1.4.1 vsan-vds02-wld01-01a
DNS: 10.1.1.1
NTP:: 10.1.1.1

Check box Export Configuration:
Next
Finish
```


# Once the supervisor is green check the API end point IP
```
Burger menu -> Supervisor Management -> Supervisors -> Control Plane Node Address
```

# Open up a Terminal session 	
```
vcf context create sup --endpoint 10.1.5.40 --insecure-skip-tls-verify --auth-type basic
#administrator@WLD.SSO/(famous VMware password twice)
vcf context use
k get nodes
```


# Trouble Shooting AVI (AKO) Pod on Supervisor cluster

(1)	vCenter
```
	ssh root@10.1.1.11   
	shell
	/usr/lib/vmware-wcp/decryptK8Pwd.py
	ssh root@10.1.5.40				#use IP/Password from above command
```

(2)	On Supervisor VM get guest cluster credentials
```
	kubectl get pods -A | grep ako
  kubctl log -n {paste from above ns and pod name}
```



