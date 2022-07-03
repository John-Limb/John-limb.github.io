---
title: Copying SSH ID on win 10 to Ubuntu (linux)
date: 2022-07-02 10:00
categories: [homelab,kubernetes,k3s,ha,ssh,ubuntu]
tags: [homelab,kubernetes,k3s,ha]
---
# Copying SSH ID on Windows 10 To Ubuntu

## Background
Typically when coyping SSH id's to remote servers / machines we would use 
```ssh-copy-id```
With windows terminal we don't have this tool but there is an easy single liner you can use.  

## Step one
Ensure you have a an SSH ID setup on your machine.  
If not create one, I reccomend using ED25519. 
```ssh-keygen -t ed25519 -C "name@mail" ```
Password protect it if you want (reccomended)  

### Why ED25519?
- It's faster to generate and verify
- It's more Secure, over older SSH Key types
- The Keys are smaller, making it easier to transfer and copy.

## Step two
Copy your new ID over to the the remote system.  
```type $env:USERPROFILE\.ssh\id_rsa.pub | ssh {IP-ADDRESS-OR-FQDN} "cat >> .ssh/authorized_keys"```

## End
You should now be able to passwordless SSH into your remote machine without issue. 