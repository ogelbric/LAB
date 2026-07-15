## Echo Server with Loadbalancer ingress

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
