# Using the VMservice VM to relocate images

## Start docker on the Photon OS VM
To see how to get a Photon VMservice VM working: https://github.com/ogelbric/LAB/blob/main/VCF-CLI/README.md
```
ssh root@10.1.14.11
tdnf install docker
init 6
systemctl start docker
docker ps
# and it returns that nothing is running
docker images
# returns no images are running

docker login docker.io
#Userid:
#PWD:
#after that comes a browser verification
```

## Images we are after from the sample app
```
# Google application yaml
wget -O sample_app.yaml https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/refs/heads/main/release/kubernetes-manifests.yaml
# Save the yaml locally
# Images we need (grep "images:" sample_app.yaml):
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/currencyservice:v0.10.5
        image: busybox:latest
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/loadgenerator:v0.10.5
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/productcatalogservice:v0.10.5
          image: us-central1-docker.pkg.dev/google-samples/microservices-demo/checkoutservice:v0.10.5
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/shippingservice:v0.10.5
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/cartservice:v0.10.5
        image: redis:alpine
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/emailservice:v0.10.5
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/paymentservice:v0.10.5
          image: us-central1-docker.pkg.dev/google-samples/microservices-demo/frontend:v0.10.5
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/recommendationservice:v0.10.5
        image: us-central1-docker.pkg.dev/google-samples/microservices-demo/adservice:v0.10.5

# Test new Harbor
docker image pull us-central1-docker.pkg.dev/google-samples/microservices-demo/currencyservice:v0.10.5
docker image ls

# should return the image local in docker
```
## Please see the Harbor install here: Need URL: https://github.com/ogelbric/LAB/tree/main/where_is(are)/Create_Harbor

## Images pull + tag + push with script 
```
# get the sample app yaml
wget -O sample_app.yaml https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/refs/heads/main/release/kubernetes-manifests.yaml
grep "image:" sample_app.yaml | awk '{print $2}' > images.txt

cat > myscript.sh <<"EOF"
#! /usr/bin/bash
p="harbor.vcf.lab/myrepo/"
for f in $(cat images.txt); do
b=$(basename $f)
echo "Processing: $f <> $b"
docker image pull $f
docker image tag $f $p$b
docker image push $p$b
docker image rm $f
docker image rm $p$b
docker image ls
done
EOF




```






