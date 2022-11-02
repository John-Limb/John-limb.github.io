---
title: Guacamole on kubetnetes
date: 2022-11-01 10:00
categories: [homelab,kubernetes,k3s,ha,cert,tls,guacamole]
tags: [homelab,kubernetes,k3s,ha,tls,ssl,cert,guacamole]
---
## Requirements
* K3s cluster or stand alone node
* Kubectl installed on your machine and configured to connect to your node / cluster
* A public domain within cloudflare
* Traekfik installed in your cluster
* Reloader installed in your cluster

## Set folder structure
We want to split this out into two parts, Guacd and Guacamole. 
Example Tree:
``` cmd
└───Manifests
    ├───Guacamole
    └───guacd
```
## Installing guacd into the cluster
Guacd is the service in the backend that will be connecting out to the other devices in the network.
To begin with we will install Guacd into the cluster as it's own set of pods. 
Start with creating a basic Namespace for the application 
#### **`guacd-ns.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: guacd
  labels:
    name: guacd
```
Now we want to create a deployment for the application, at the time of writing this version 1.4.0 was the latest version. I reccomend fixing to a certain version and not using the latest tag for deployments. 
#### **`guacd-deployment.yaml`**
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
   name: guacd
   namespace: guacd
spec:
  selector:
    matchLabels:
      app: guacd
  replicas: 1 # < Set to as many replicas as you desire
  template:
     metadata:
       labels:
         app: guacd
     spec:
        containers:
         - name: guacd
           image: guacamole/guacd:1.4.0
           ports:
           - containerPort: 4822
```
Create a service for the guacamole service to talk to later on.
#### **`guacd-Service.yaml`**
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: guacd-svc
  namespace: guacd
  labels:
    app: guacd
spec:
  selector:
    app: guacd
  ports:
  - port: 4822
    protocol: TCP 
```
Apply the manifests and check that the deployment has spun up
``` bash
kubectl get pods -o wide -n guacd
```
## Now to deploy Guacamole front end
Create a new namespace for guacamole
### **`Guacamole-namespace.yaml`**
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: guacamole
```
Create a config map for the deployment to look at for configuration. We are using a config map keep the server details in, sadly it is in XML format.
Set yourself a login and a password, encode it using an md5 tool. 
### **`Guacamole-configmap.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: guacamole-conf
  namespace: guacamole
data:
  user-mapping.xml: |
    <user-mapping>
    <authorize 
    username="myaccount" 
    password="apasswordinmd5encoding" 
    encoding="md5">
    <connection name="aservername">
    <protocol>ssh</protocol>
    <param name="hostname">serveriphere</param>
    <param name="port">22</param>
    <param name="username">myadminaccont</param>
    </connection>
    <connection name="anotherserver">
    <protocol>rdp</protocol>
    <param name="hostname">anotherip</param>
    <param name="port">3389</param>
    <param name="username">Administrator</param>
    <param name="ignore-cert">true</param>
    </connection>
    </authorize>
    </user-mapping>
```
Create a deployment for the guacamole service
### **`Guacamole-Deployment.yaml`**
```yaml
--- 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guacamole
  namespace: guacamole
spec:
  selector:
    matchLabels:
      app: guacamole
  replicas: 1
  strategy:
    type: RollingUpdate
  revisionHistoryLimit: 6 
  template:
    metadata:
      labels:
        app: guacamole
    spec:
      containers:
      - name: guacamole
        image: guacamole/guacamole:1.4.0 #< Change this to match the version of GuacD you are using
        env:
          - name: GUACD_HOSTNAME
            value: "guacd-svc.guacd" # < Change the guacd service dns name if you changed it in the guacd-service yaml file
          - name: GUACD_PORT
            value: "4822"
          - name: GUACAMOLE_HOME
            value: /etc/guacamole/
        resources:
          limits:
            memory: "512Mi" # < Adjust memory and cpu limits here if needed
            cpu: "800m"
        ports:
        - name: http
          containerPort: 8080 
        volumeMounts:
          - name: guacamole-conf
            mountPath: /etc/guacamole/
        livenessProbe:
          httpGet:
            path: /guacamole/
            port: 8080
          initialDelaySeconds: 45 # < Adjust this if your pods take a while to spin up.. 45 seconds should be plenty
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
              path: /guacamole/
              port: 8080
          initialDelaySeconds: 45 # < Adjust this if your pods take a while to spin up.. 45 seconds should be plenty
          timeoutSeconds: 40
      volumes:
        - name: guacamole-conf
          configMap:
            name: guacamole-conf
```
Now we can ceate a service for the deploment
### **`Guacamole-Service.yaml`**
``` yaml
---
apiVersion: v1
kind: Service
metadata:
  name: guacamole-svc
  namespace: guacamole
  labels:
    app: guacamole
spec:
  selector:
    app: guacamole
  ports:
  - port: 8080
    protocol: TCP
```
And now create and ingress for the service above. We are going to use a middleware also to auto add the /guacmole onto the url so you don't run into an apache 404 error when targetting your URL.
### **`Guacamole-ingress.yaml`**
```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: guacamole
  namespace: guacamole
  annotations: 
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`guacamole.domain.tld`)
      kind: Rule
      services:
        - name: guacamole-svc
          port: 8080
      middlewares:
        - name: prepend-path-guacamole
  tls:
    secretName: prod-guacamole-domain-tld
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: prepend-path-guacamole
spec:
  addPrefix:
    prefix: /guacamole
---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: guacamole.domain.tld
      namespace: guacamole
    spec:
      secretName: prod-guacamole-domain-tld
      issuerRef:
        name: letsencrypt-production
        kind: ClusterIssuer
      commonName: "guacamole.domain.tld"
      dnsNames:
      - "guacamole.domain.tld"
```
Configure a DNS name on your DNS server etc and you should be greeted with the guacamole login page. 