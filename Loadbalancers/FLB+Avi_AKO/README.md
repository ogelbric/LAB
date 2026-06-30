# Trying FLB with Avi + AKO in the guest cluster 

```
Deploying a vSphere Kubernetes Service (VKS) cluster and configuring the Avi Kubernetes Operator (AKO) relies heavily on your underlying networking configuration (NSX or VDS).
In most VCF and VKS environments, you deploy AKO into your VKS cluster using Helm to manage Layer 4 and/or Layer 7 workloads.
Follow these steps to deploy and configure AKO in your VKS environment:
Step 1: Meet the PrerequisitesBefore installing AKO, ensure the following core infrastructure requirements are in place:
Avi Controller: Deployed and operational, with an active connection to your vCenter Server instance.
IPAM/DNS Profiles: Configured on your Avi Controller to handle your front-end VIP networks and DNS mapping.
VKS Cluster:Provisioned and active within a vSphere Supervisor Namespace.
Step 2: Extract Your Values ConfigurationTo connect AKO to your Avi infrastructure, you will need to map specific parameters in your values.yaml file.
To configure AKO for an NSX network topology, use settings similar to the following:
ControllerSettings.controller
Host: IP address or FQDN of your Avi Controller.ControllerSettings.cloud
Name: The name of your vCenter cloud connector configured on the Avi Controller.L7Settings.service
Type: Set to NodePort (or NodePortLocal) so Service Engines (SEs) can forward traffic to pods while continuing to expose internal services as ClusterIP.vipNetwork
List: Set to the specific IP network/subnet used for your Avi Virtual Services.nsxt
T1LR: Full path to your Tier-1 Logical Router (e.g., /infra/tier-1s/Avi-T1).
Step 3: Deploy AKO with HelmOnce your values.yaml is configured, install the AKO operator using Helm via OCI:
Set your kubectl context to your targeted VKS cluster.Create a dedicated namespace for the operator:
bash kubectl create namespace avi-system
Use code with caution.
Run the helm install command to deploy AKO. Substitute your specific Avi Controller IP and credentials below:
```
bash helm install --generate-name \
  oci://projects.packages.broadcom.com/ako/helm-charts/ako \
  --version 1.13.3 \
  -f values.yaml \
  --set ControllerSettings.controllerHost=10.11.10.151 \
  --set avicredentials.username=admin \
  --set avicredentials.password=YourAviPassword \
  --namespace=avi-system
```
Use code with caution.
Step 4: Verify the DeploymentConfirm that the AKO operator pods are running properly within your VKS cluster by using the following command:bashkubectl get pods -n avi-system
Use code with caution.
Step 5: Leverage LoadBalancerClass for VKS ServicesIf your vSphere Supervisor is handling Layer 4 traffic across multiple clusters, you must specify the loadBalancerClass in your Kubernetes service specification so that AKO manages the exact VKS service rather than the Supervisor cluster:yamlapiVersion: v1
```
kind: Service
metadata:
  name: my-vks-service
spec:
  type: LoadBalancer
  loadBalancerClass: avi-lb # Matches your AKO setup
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```
