# NFS
1. Setup NFS server
2. Provide ability to mount file systems

**Server IP:** 192.168.31.31  
**Client IP:** 192.168.31.101  
**Read-only resource:** /usr/home/public  
**Read-write resource:** /usr/home/private  

--- 
### Setup server   
Install NFS  
```shell script
$ sudo apt install nfs-kernel-server
```

Create directories  
```shell script
$ sudo mkdir -p /usr/home/public
$ sudo mkdir -p /usr/home/private
```

Configure owners and rights
```shell script
$ sudo chown nobody:nogroup /usr/home/public
$ sudo chown nobody:nogroup /usr/home/private

$ sudo chmod 777 /usr/home/public
$ sudo chmod 777 /usr/home/private
```

Configure file systems that are accessible for NFS clients
```shell script
$ sudo nano /etc/exports
```
```text
/usr/home/public     192.168.31.0/24(ro,sync,no_subtree_check)
/usr/home/private    192.168.31.101(rw,sync,no_subtree_check)
```
> where `192.168.31.101` is client IP

```shell script
$ sudo exportfs -a
$ sudo systemctl restart nfs-kernel-server
```

### Setup client 
Install NFS client
```shell script
$ sudo apt install nfs-common
```

Create directories accordingly
```shell script
$ sudo mkdir -p /usr/home/public_client
$ sudo mkdir -p /usr/home/private_client
```

Mount file systems
```shell script
$ sudo mount 192.168.31.31:/usr/home/public /usr/home/public_client
$ sudo mount 192.168.31.31:/usr/home/private /usr/home/private_client
```
Check result
```shell script
$ df -h
```
Unmount if needed
```shell script
$ sudo umount /usr/home/public_client
```

### Check access

Try to create file in the folder with read-only access
```shell script
$ touch /usr/home/public_client/test.txt
```
The following message will be displayed:  
```text
touch: cannot touch '/usr/home/public_client/test.txt': Read-only file system
```
Try to create file in the folder with read-write access
```shell script
$ touch /usr/home/private_client/test.txt
```
Check if created
```shell script
$ ls /usr/home/private_client/
```
--- 
![Results](/imgs/lab5.png)
