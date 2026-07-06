# Create Harbor as Supervisor service

## Download the harbor bits
```
Go to support.broadcom.com
Serach for: Supervisor Services
Select Harbor latest version
Download: harbor-sys.yaml and harbor-data-values.yaml
```


## Install the Harbor supervisor service 

```
Burger Menu -> Supervisor Management -> Services -> Add new Service
Upload the Harbor Supervisor Servcies
```

## Configure the harbor data values file 
```
cp supervisor-service-harbor-data-values-v2.14.3.yml t.yaml

sed 's/hostname: yourdomain.com/hostname: harbor.vcf.lab/' t.yaml > /tmp/a1
sed 's/enableNginxLoadBalancer: false/enableNginxLoadBalancer: true/' /tmp/a1 > /tmp/a2
sed 's/enableContourHttpProxy: true/enableContourHttpProxy: false/' /tmp/a2 > /tmp/a3
sed 's/enableContourHttpProxy: true/enableContourHttpProxy: false/' /tmp/a3 > /tmp/a4
sed 's/insert-storage-class-name-here/vsan-default-storage-policy/' /tmp/a4 > /tmp/a5
cp /tmp/a5 ~/Downloads/a5.yaml


```
