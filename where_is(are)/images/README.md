# Using the VMservice VM to relocate images

## Start docker on the Photon OS VM
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
https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/refs/heads/main/release/kubernetes-manifests.yaml
# Save the yaml locally
# Images we need:
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


docker image pull us-central1-docker.pkg.dev/google-samples/microservices-demo/currencyservice:v0.10.5
# works

We need a harbor

```

