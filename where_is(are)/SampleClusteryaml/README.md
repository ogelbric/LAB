Sample Cluster yaml with addons

```
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-headlamp
  namespace: demo1 # Your vSphere namespace
spec:
  addonRef:
    name: headlamp
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks1
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: workload-vsphere-vks1-headlamp
  namespace: demo1 # Your vSphere namespace
spec:
  values:
    gatewayApi:
      enabled: true
      gateway:
        className: istio
        create: true
        name: headlamp-gateway
    hostname: headlamp.192.168.104.106.sslip.io
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-cert-manager
  namespace: demo1 # Your vSphere namespace
spec:
  addonRef:
    name: cert-manager
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks1
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-prom
  namespace: demo1 # Your vSphere namespace
spec:
  addonRef:
    name: prometheus
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks1
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: workload-vsphere-vks1-prometheus
  namespace: demo1 # Your vSphere namespace
spec:
  values:
    deploycomponents:
      kube-state-metrics: true
      node-exporter: true
      pushgateway: false
      prometheus: true
      alertmanager: false
      prometheus-operator: true
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-istio
  namespace: demo1 # Your vSphere namespace
spec:
  addonRef:
    name: istio
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks1
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: workload-vsphere-vks1-istio
  namespace: demo1  # Your vSphere namespace
spec:
  values:
    istio:
      namespace: "istio-system"
      ambientMode:
        enabled: false
      istioCNI:
        enabled: false
      gateways:
        egress:
          enabled: false
        ingress:
          enabled: true
          namespace: "istio-system"
          autoscaling:
            enabled: true
            minReplicas: 1
            maxReplicas: 5
      pilot:
        replicas: 1
        autoscaling:
          enabled: true
          minReplicas: 1
          maxReplicas: 2
      meshConfig:
        enableTracing: false
        accessLogFile: ""
        meshID: "workload-vsphere-vks1"
---
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: workload-vsphere-vks1
  namespace: demo1
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 240.0.0.0/20
    serviceDomain: cluster.local
    services:
      cidrBlocks:
      - 240.1.0.0/20
  topology:
    classRef:
      name: builtin-generic-v3.7.0
      namespace: vmware-system-vks-public
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
      replicas: 1
      variables:
        overrides:
        - name: vmClass
          value: best-effort-large
    variables:
    - name: storageClass
      value: tanzu
    - name: vmClass
      value: best-effort-medium
    - name: volumes
      value:
      - capacity: 30Gi
        mountPath: /var/lib/containerd
        name: containerd
        storageClass: tanzu
      - capacity: 30Gi
        mountPath: /var/lib/kubelet
        name: kubelet
        storageClass: tanzu
    - name: kubernetes
      value:
        certificateRotation:
          enabled: true
          renewalDaysBeforeExpiry: 90
        security:
          podSecurityStandard:
            audit: privileged
            auditVersion: latest
            deactivated: false
            enforce: privileged
            enforceVersion: latest
            warn: privileged
            warnVersion: latest
    - name: vsphereOptions
      value:
        persistentVolumes:
          availableStorageClasses:
          - tanzu
          defaultStorageClass: tanzu
    - name: osConfiguration
      value:
        ntp:
          servers:
          - ntp.broadcom.net
        sshd:
          banner: '---AUTHORIZED ACCESS ONLY--- Use of this system is restricted to
            authorized users for authorized activities only. '
    version: v1.35.2---vmware.1-vkr.3
    workers:
      machineDeployments:
      - class: node-pool
        metadata:
          annotations:
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "5"
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "1"
            run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
        name: node-pool-1
```

Sample cluster yaml with containerd mount point

```
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster01-new
  namespace: namespace1000
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
        - 192.168.156.0/20
    services:
      cidrBlocks:
        - 10.96.0.0/12
    serviceDomain: cluster.local
  topology:
    class: builtin-generic-v3.6.0
    classNamespace: vmware-system-vks-public
    version: v1.33.6---vmware.1-fips-vkr.2
    variables:
      - name: vsphereOptions
        value:
          persistentVolumes:
            defaultStorageClass: k8storage
      - name: kubernetes
        value:
          certificateRotation:
            enabled: true
            renewalDaysBeforeExpiry: 90
      - name: vmClass
        value: guaranteed-xsmall
      - name: storageClass
        value: k8storage
      - name: volumes
        value:
        - name: containerd
          mountPath: /var/lib/containerd
          storageClass: k8storage
          capacity: 15Gi
        - name: kubelet-override
          mountPath: /var/lib/kubelet
          storageClass: k8storage
          capacity: 10Gi          
        - name: log
          mountPath: /var/log
          storageClass: k8storage
          capacity: 10Gi          
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
        - class: node-pool
          name: cluster21-worker
          replicas: 1
```

