


## SSH into proxmox lxc


By default a Proxmox LXC container allows root login only with public key authentication. To login to a container with username/password login to your Proxmox host and attach to the container with the following command.

```
proxmox$ lxc-attach --name 101
```


Next, go to the lxc console from proxmox webUI.

Open sshd_config and change the line `PermitRootLogin without-password` to `PermitRootLogin yes`. Exit nano with Ctrl+X and save changes with y and ENTER.

```
lxc$ nano /etc/ssh/sshd_config
```

Finally, Restart ssh service for the changes to take effect


```
lxc$ service ssh restart
```

- https://pmsl.com.ng/ssh-into-a-proxmox-lxc-container/
