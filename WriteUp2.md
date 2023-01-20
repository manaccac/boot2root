# Boot2Root 42 Project - Write Up 2

## Step 1: acces VM's booting script

While loading the VM hold the key 
```
ctrl + alt + f1
```

## Step 2: Change it

Now all you need to do is to write 
```
live /boot/ init=/bin/sh
```
press enter and you are root 

```
# whoami
root
```