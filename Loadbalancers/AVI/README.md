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
admin/(famouse password)
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




