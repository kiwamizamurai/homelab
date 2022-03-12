

## apt-get update fails


```
TASK ERROR: command 'apt-get update' failed: exit code 100
```

solution is as follows

```
root@proxmox:~# cat /etc/apt/sources.list.d/pve-enterprise.list
# deb https://enterprise.proxmox.com/debian/pve buster pve-enterprise
```

- https://stackoverflow.com/questions/61998378/proxmox-apt-get-update-fails