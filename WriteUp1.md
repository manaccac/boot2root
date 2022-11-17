# Boot2Root 42 Project - Write Up 1

## Step 1: find the VM's IP

First, find your local ip so you scan it

```
on Windows :
>ipconfig
   IPv4 Address. . . . . . . . . . . : 192.168.1.45
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.255
on Macs :
    Maxime je te laisse remplir ca
```

Now we know our local ip, so we'll scan it using nmap ans after a few minutes we end up with :

```
>nmap 192.168.1.0/24
```