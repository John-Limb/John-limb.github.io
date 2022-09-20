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
## Monitoring Servers.  
These are the most common and obvious choices for monitoring Kubernetes.  
- Prometheus
- Graphana

## What we will need
- A working Kubernetes cluster
- Helm on your local machine
- Kubectl on your local machine

