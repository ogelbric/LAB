# Create Git

## Load Helm CLI to jump box (need URL here!!!)
```
sudo tdnf install git
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
sudo ./get_helm.sh
sudo chmod 755 /usr/local/bin/helm
helm version
```

## Loginto the guest cluster 
```
FYI - Guest cluster needs to be created with default storage policy!!!
vcf context create  --endpoint https://10.1.4.41 --username administrator@WLD.SSO --insecure-skip-tls-verify --auth-type basic --workload-cluster-name kubernetes-cluster-5smg --workload-cluster-namespace namespace1000

vcf context use
? Select a context guest88:k8-orf-cluster1
kubectl get nodes 

NAME                                                        STATUS   ROLES           AGE     VERSION
k8-orf-cluster1-k8-orf-cluster1-np-w29q-m6z4r-w5k6g-4rjjm   Ready    <none>          7h45m   v1.35.5+vmware.1
k8-orf-cluster1-k8-orf-cluster1-np-w29q-m6z4r-w5k6g-4rtg6   Ready    <none>          7h45m   v1.35.5+vmware.1
k8-orf-cluster1-k8-orf-cluster1-np-w29q-m6z4r-w5k6g-57dhl   Ready    <none>          7h45m   v1.35.5+vmware.1
k8-orf-cluster1-k8-orf-cluster1-np-w29q-m6z4r-w5k6g-992xf   Ready    <none>          7h45m   v1.35.5+vmware.1
k8-orf-cluster1-k8-orf-cluster1-np-w29q-m6z4r-w5k6g-f8rdv   Ready    <none>          7h45m   v1.35.5+vmware.1
k8-orf-cluster1-lxsfw-p2l8n                                 Ready    control-plane   7h47m   v1.35.5+vmware.1
root@orfvm [ ~ ]# 
```

## Install Git
```
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
kubectl create namespace git
kubectl label --overwrite ns git pod-security.kubernetes.io/enforce=privileged
helm install gitea gitea-charts/gitea -n git

NOTES:
1. Get the application URL by running these commands:
  echo "Visit http://127.0.0.1:3000 to use your application"
  kubectl --namespace default port-forward svc/gitea-http 3000:3000

so this works.... now we need an IP...and change the values file for type loadbalancer

helm uninstall gitea
helm list --all-namespaces
```


## Vaules file
```
cat > values-gitea.yaml <<"EOF"
podSecurityContext:
  fsGroup: 0
service:
  http:
    type: LoadBalancer
    loadBalancerIP: 10.1.5.49
    port: 443
  ssh:
    type: LoadBalancer
    loadBalancerIP: 10.1.5.48
    port: 22
    ClusterIP:
EOF
```

## Install git ea with values file
```
kubectl create namespace git
helm install gitea gitea-charts/gitea --values values-gitea.yaml -n git
```


## Test ingress with Echo Server (optional) 

```
kubectl label --overwrite ns es pod-security.kubernetes.io/enforce=privileged
kubectl run echoserver --image=gcr.io/google-containers/echoserver:1.10 --port=8080 -n es

cat <<EOF > loadbalancer.yaml
---
kind: Service
apiVersion: v1
metadata:
  name: loadbalanced-service
spec:
  selector:
    run: echoserver
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
EOF
kubectl apply -f loadbalancer.yaml -n es
kubectl get service loadbalanced-service -n es
```

## Did not have to go down this road since I was able to pull the images
## This option needs to be expored more in locked down environments

```
#pulling the images
help pull gitea-charts/gitea
helm registry login -u myuser localhost:5000
Password:
Login succeeded
helm registry logout localhost:5000
helm push mychart-0.1.0.tgz oci://localhost:5000/helm-charts
$ helm install myrelease oci://localhost:5000/helm-charts/mychart --version 0.1.0
$ helm upgrade myrelease oci://localhost:5000/helm-charts/mychart --version 0.2.0
```


  
