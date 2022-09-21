---
title: K3S cluster Monitoring
date: 2022-07-01 10:00
categories: [homelab,kubernetes,k3s,ha]
tags: [homelab,kubernetes,k3s,ha]
---
# Monitoring K3S cluster Nodes and hardware.  
## Kubernetes Stack
Current stack consists of:
- 3 K3S Master,ETCD,Control plane servers (See other page about creating and configuring those.)  
- 2 K3S Worker nodes

## What we will need
- A working Kubernetes cluster
- Helm on your local machine
- Kubectl on your local machine
- Traefik installed on your cluster

We will be using [Kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)\
This will give us all we need, including graphs to apply monitoring to the cluster.

### Adding the Helm Repo
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
Update
```shell
helm repo update
```
### Create monitoring namespace
```shell
kubectl create ns monitoring
```
### Create grafana credentials
I would advise doing this in linux or WSL if you are windows, you can SCP or copy the files out later to the desired folder of choice. 
```shell
echo -n 'Anadminuser' > ./admin-user # change your username
echo -n 'Ap@ssword!' > ./admin-password # change your password
```
Create a secret
```shell
 kubectl create secret generic grafana-admin-credentials --from-file=./admin-user --from-file=admin-password -n monitoring
```
This will create a generic secret for the grafana admin account. 

Test that the credential has been saved
```shell
kubectl describe secret -n monitoring grafana-admin-credentials
```
If all is ok, I would advise removing the credentials from your local machine, you don't want them to be synced into some form of cloud drive or github repo etc. 

### installing the stack
```shell
helm install -n monitoring prometheus prometheus-community/kube-prometheus-stack -f values.yaml
```
Take the values of the file below - Change the IP's listed to the IP's of your master kubernetes nodes. 
```yaml
fullnameOverride: prometheus

defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    configReloaders: true
    general: true
    k8s: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeScheduler: true
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true

alertmanager:
  fullnameOverride: alertmanager
  enabled: true
  ingress:
    enabled: false

grafana:
  enabled: true
  fullnameOverride: grafana
  forceDeployDatasources: false
  forceDeployDashboards: false
  defaultDashboardsEnabled: true
  defaultDashboardsTimezone: utc
  serviceMonitor:
    enabled: true
  admin:
    existingSecret: grafana-admin-credentials
    userKey: admin-user
    passwordKey: admin-password

kubeApiServer:
  enabled: true

kubelet:
  enabled: true
  serviceMonitor:
    metricRelabelings:
      - action: replace
        sourceLabels:
          - node
        targetLabel: instance

kubeControllerManager:
  enabled: true
  endpoints: # ips of servers 
    - 192.168.1.8 #change me
    - 192.168.1.9 #change me
    - 192.168.1.10 #change me

coreDns:
  enabled: true

kubeDns:
  enabled: false

kubeEtcd:
  enabled: true
  endpoints: # ips of servers
    - 192.168.1.8 #change me
    - 192.168.1.9 #change me
    - 192.168.1.10 #change me
  service:
    enabled: true
    port: 2381
    targetPort: 2381

kubeScheduler:
  enabled: true
  endpoints: # ips of servers
    - 192.168.1.8 #change me
    - 192.168.1.9 #change me
    - 192.168.1.10 #change me

kubeProxy:
  enabled: true
  endpoints: # ips of servers
    - 192.168.1.8 #change me
    - 192.168.1.9 #change me
    - 192.168.1.10 #change me

kubeStateMetrics:
  enabled: true

kube-state-metrics:
  fullnameOverride: kube-state-metrics
  selfMonitor:
    enabled: true
  prometheus:
    monitor:
      enabled: true
      relabelings:
        - action: replace
          regex: (.*)
          replacement: $1
          sourceLabels:
            - __meta_kubernetes_pod_node_name
          targetLabel: kubernetes_node

nodeExporter:
  enabled: true
  serviceMonitor:
    relabelings:
      - action: replace
        regex: (.*)
        replacement: $1
        sourceLabels:
          - __meta_kubernetes_pod_node_name
        targetLabel: kubernetes_node

prometheus-node-exporter:
  fullnameOverride: node-exporter
  podLabels:
    jobLabel: node-exporter
  extraArgs:
    - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
    - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$
  service:
    portName: http-metrics
  prometheus:
    monitor:
      enabled: true
      relabelings:
        - action: replace
          regex: (.*)
          replacement: $1
          sourceLabels:
            - __meta_kubernetes_pod_node_name
          targetLabel: kubernetes_node
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 2048Mi

prometheusOperator:
  enabled: true
  prometheusConfigReloader:
    resources:
      requests:
        cpu: 200m
        memory: 50Mi
      limits:
        memory: 100Mi

prometheus:
  enabled: true
  prometheusSpec:
    replicas: 1
    replicaExternalLabelName: "replica"
    ruleSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
    retention: 6h
    enableAdminAPI: true
    walCompression: true

thanosRuler:
  enabled: false
```
This should install the stack, there will will be no ingress yet, you will have to port forward for testing
```shell
kubectl port-forward -n monitoring POD NAME HERE 52222:3000
```
### Setting up an ingress for grafana
For a basic ingress route with a cert applied [see here for cert creation](https://docs.limb.network/posts/certs-in-k3s/)\
Note, you may not have ingress class annotations depending on how you have your ingress setup.
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations: 
    kubernetes.io/ingress.class: traefik-external 
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`grafana.int.domain.tld`)
      kind: Rule
      services:
        - name: grafana
          port: 80
          sticky:
            cookie:
              httpOnly: true
              name: grafana
              secure: true
              sameSite: none
  tls:
    secretName: prod-grafana-int-domain-tld
---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: grafana.int.domain.tld
      namespace: monitoring
    spec:
      secretName: prod-grafana-int-domain-tld
      issuerRef:
        name: letsencrypt-production
        kind: ClusterIssuer
      commonName: "grafana.int.domain.tld"
      dnsNames:
      - "grafana.int.domain.tld"
```