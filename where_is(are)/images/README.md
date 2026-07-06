# Using the VMservice VM to relocate images

33 Start docker on the Photon OS VM
```
ssh root@10.1.14.11
tdnf install docker
init 6
systemctl start docker
docker ps
# and it returns that nothing is running
docker images
# returns no images are running






```

