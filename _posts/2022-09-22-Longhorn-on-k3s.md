---
title: Lonhorn Storage on K3s
date: 2022-09-22 10:00
categories: [homelab,kubernetes,k3s,ha,cert,tls,longhorn,storage,persistent]
tags: [homelab,kubernetes,k3s,ha,tls,ssl,cert,storage,longhorn,persistent]
---
## Requirements
* K3s cluster or stand alone node
* Kubectl installed on your machine and configured to connect to your node / cluster
* A public domain within cloudflare
* Helm installed on your machine


## Setting up
Starting off by adding some separate volumes into the two worker nodes I have within my cluster.
These will be two 100GB volumes for now but will be able to expand in the future if needs be.
### Creating the volume
Grab the disk ID, it will be something like /DEV/SDB etc. 
```shell
lsblk
```
Create a file system on the disk
```shell
sudo mkfs -t ext4 /dev/sdb
```
### Mount the Volume
You will need to make a folder on Worker node like "Longhorn" etc. 
```shell
sudo mkdir /mnt/longhorn
```
Mount the volume with 
```shell
sudo mount /dev/sdb /mnt/longhorn
```
You should then be able to test writing to the folder with a file. 
```shell
sudo touch /mnt/longhorn/test.txt
```

You will need to make the volume auto mount on boot up. 
Grab the disk UUID 
```shell
sudo blkid
```
Copy that out and put it in /etc/fstab

```shell
sudo nano /etc/fstab
```
Reboot your node and check the volume has auto mounted 
```shell
lsblk
```
You should see that your volume has mounted into the folder you created above. 

## installing open ISCSI 
You will need to install open ISCSI so longhorn can interact with storage on other nodes
```shell
sudo apt-get install open-iscsi
```
## installing longhorn
You will need to install the longhorn repo into helm
```shell 
helm repo add longhorn https://charts.longhorn.io
```
run the installer with helm
```shell
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```
Give the deployment 2 minutes or so to setup etc..\

## Exposing the dashboard
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: longhorn-dashboard
  namespace: longhorn-system
  annotations: 
    kubernetes.io/ingress.class: traefik-external # You may not have this, depending on how you have your ingress setup
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`longhorn.int.domain.tld`) #Add your DNS name here
      kind: Rule
      services:
        - name: longhorn-frontend
          port: 80
  tls:
    secretName: prod-longhorn-int.domain.tld #Match this to the name of your cert created below
---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: longhorn.int.domain.tld #Give a proper name for your cert here - like DNS name
      namespace: longhorn-system
    spec:
      secretName: prod-longhorn-int.domain.tld #Give a proper name for your cert here - like DNS name
      issuerRef:
        name: letsencrypt-production
        kind: ClusterIssuer
      commonName: "longhorn.int.domain.tld" #Add your DNS name here
      dnsNames:
      - "longhorn.int.domain.tld" #Add your DNS name here
```
You should now have a dashboard at the DNS address you created erlier. 