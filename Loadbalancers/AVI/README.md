# AVI Loadbalancer


## Download the AVI OVA
```
http://support.broadcom.com
Downloads
Search fro AVI
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




