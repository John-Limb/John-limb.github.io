---
title: Auto Upgrading Pihole With Ansible
date: 2022-07-02 11:00
categories: [homelab,ha,ssh,ubuntu]
tags: [homelab,Ansible,Pihole]
---

# Auto Updating Pi-hole with Ansible

# Requirements
- Pihole server with SSH passwordless auth enabled
- Ansible

## why?
Keeping pihole up to date means we are getting the latest bugfixes and features.  
We don't always remember to upgrade and patch services so why not let automation do that for us.  
## What is needed?
A simple YAML file, that we can use with Ansible (See other post on getting AWX running within Kubernetes).  
Here is the YAML file I wrote in order to patch Pihole on a monthly basis. Granted they might not release that frequently but it's ways better to keep it up to date. 

```yaml 
--- 
- 
  become: false
  hosts: "*"
  tasks: 
    - 
      name: "Look For Pihole upgrade"
      register: shell_output
      shell: "pihole -up"
    - 
      debug: var=shell_output.stdout_lines
```
Telling Ansible to open a shell on the remote server and run the pihole -up command. And parse the output of the shell output back into the ansible console.