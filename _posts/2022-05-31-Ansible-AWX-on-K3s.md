---
title: Ansible AWX on K3s
date: 2022-05-31 10:00
categories: [homelab,kubernetes,k3s,ha,ansible,awx]
tags: [homelab,kubernetes,k3s,ha,ansible,awx]
---
# Ansible AWX on K3S
Ansible AWX provides a web interface and REST API for Ansible.  
[Ansible AWX Git Repo](https://github.com/ansible/awx)  
It provides us with a task engine which makes configuring and setting up repeated tasks like updates a breeze. 

## Requirements
* K3s cluster or stand alone node
* Kubectl installed on your machine and configured to connect to your node / cluster

## Setup tools to build components
You will need to install JQ and git and also make onto your system. 
```bash
sudo apt install jq git make
```
Now you can clone the AWX - operator repo to your local machine
```bash
git clone https://github.com/ansible/awx-operator.git
```
Create a namespace for AWX
```bash
export NAMESPACE=awx
kubectl create ns ${NAMESPACE}
```
Now change into the directory of repo you cloned erlier
```
cd awx-operator/
```
Fetch the latest release tag for the AWX 
``` bash
RELEASE_TAG=`curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4`
echo $RELEASE_TAG
```
Now we can deploy the operator into the namespace we made erlier
```bash
export NAMESPACE=awx
make deploy
```
Wait a few moments and the operator should be deployed into the namespace
```bash
kubectl get pods -o wide -n awx
```
## Create a persistant volume claim 
We now need to create a PVC for the persistant data Ansible is going to use.   
If we are using a single node we can create it using this PVC yaml file
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ansible-pvc
  namespace: awx
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 8Gi

```
This will create a persistant volume on the local node of 8GB for the pod.  
If you are using a cluster you will want to taint your nodes and ensure you add the right selectors to your PVC YAML file. this is so that IF the pods needs to be re-scheduled it will be on the correct node.  
  
    
Now we can apply the PVC with:  
```bash
kubectl apply -f public-static-pvc.yaml -n awx
```
You should now see the PVC is pending with:  
```bash
kubectl get pvc -n awx
```
In an editor, create a new file called "Awx-Instance-deploy.yaml"
```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: nodeport
  projects_persistence: true
  projects_storage_access_mode: ReadWriteOnce
  web_extra_volume_mounts: |
    - name: static-data
      mountPath: /var/lib/projects
  extra_volumes: |
    - name: static-data
      persistentVolumeClaim:
        claimName: ansible-pvc
```
For now, we are going to deploy the pod as a node port. We can always change this and put it behind an ingress in the future and apply TLS.  
  
You can now apply the file
```bash
kubectl apply -f Awx-Instance-deploy.yaml -n awx
```
This will deploy the AWX pod into the AWX namespace.  
To check what is happening you can issue a ```watch``` command.  
```bash
watch kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator" -n awx
```
You should also see some more PVC's have been created
```
kubectl  get pvc
```
If you need to check the logs:  
```bash
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```
All of the pods should be spun up. To check:
```bash
kubectl get pods -o wide -n awx
```
If you ever need to check the logs of the pods:
```bash
kubectl -n awx  logs deploy/awx -c awx-task
kubectl -n awx  logs deploy/awx -c redis
kubectl -n awx  logs deploy/awx -c awx-web
kubectl -n awx  logs deploy/awx -c awx-ee
```
## Accessing the web interface
To grab the port of the web interface
```
kubectl get service -n awx
```
This should show that port is 30080 that is the default for AWX within kubernetes.  
You can always change it within the deploy yaml file you created erlier. 