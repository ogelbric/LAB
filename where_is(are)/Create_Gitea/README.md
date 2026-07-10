# Create Git

## Load Helm cli to jump box (need URL here!!!)
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

## Loginto the guest cluster 
```
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
helm install gitea gitea-charts/gitea

NOTES:
1. Get the application URL by running these commands:
  echo "Visit http://127.0.0.1:3000 to use your application"
  kubectl --namespace default port-forward svc/gitea-http 3000:3000

so this works.... now we need an IP...and change the values file for type loadbalancer

helm uninstall gitea
helm list --all-namespaces
```

## test
```
cat > values-gitea.yaml <<"EOF"
persistence:
  create: false
  claimName: nfs-test

postgresql-ha:
  enabled: false

postgresql:
  enabled: true

http:
    type: ClusterIP
    port: 3000
    clusterIP: None
    loadBalancerIP: 10.1.4.45
EOF

kubectl create namespace git
helm install gitea gitea-charts/gitea --values values-gitea.yaml -n git
```


# test 3

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


  
