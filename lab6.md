# SAMBA
1. Install and configure Samba server
2. Enable access to shared resources

| user     | hidden    | home      | pub       | incoming  |
|----------|-----------|-----------|-----------|-----------|
| mercury  |    RW     |    RW     |    R      |    RW     |
| venus    |    R      |    RW     |    RW     |    RW     |
| guest    |    -      |    -      |    R      |    RW     |

----

## Install Samba
```shell script
$ sudo apt install samba
```
## Configure access
```shell script
$ sudo nano /etc/samba/smb.conf
```
```text
[global]
workgroup = WORKGROUP
security = user
map to guest = bad user
wins support = no
dns proxy = no

[hidden]
path = /var/samba/hidden
guest ok = no
browsable = no
writable = yes
valid users = @smbgrp

[home]
path = /var/samba/home
guest ok = no
browsable = yes
writable = yes
valid users = @smbgrp

[pub]
path = /var/samba/pub
guest ok = yes
browsable = yes
writable = yes

[incoming]
path = /var/samba/incoming
guest ok = yes
browsable = yes
writable = yes
```

```shell script
$ service smbd restart
```

```shell script
$ iptables -A INPUT -p tcp -m tcp --dport 445 -s 192.168.31.0/24 -j ACCEPT
$ iptables -A INPUT -p tcp -m tcp --dport 139 -s 192.168.31.0/24 -j ACCEPT
$ iptables -A INPUT -p udp -m udp --dport 137 -s 192.168.31.0/24 -j ACCEPT
$ iptables -A INPUT -p udp -m udp --dport 138 -s 192.168.31.0/24 -j ACCEPT
```

```shell script
$ sudo groupadd smbgrp
$ sudo usermod -aG smbgrp venus
$ sudo usermod -aG smbgrp mercury

$ sudo chgrp smbgrp /var/samba/home/
$ sudo chgrp smbgrp /var/samba/hidden/

$ sudo smbpasswd -a venus
$ sudo smbpasswd -a mercury

$ sudo chmod -R 0775 /var/samba/home/
$ sudo chown -R mercury:smbgrp /var/samba/home/

$ sudo chmod -R 0755 /var/samba/pub/
$ sudo chown -R venus:smbgrp /var/samba/pub/

$ sudo chmod -R 0777 /var/samba/incoming/
```

---
### Results  
File tree for `venus` user  
![File tree for venus](/imgs/lab6_venus.png)  
Read-only access to hidden folder for `venus` user  
![Read-only for venus](/imgs/lab6_venus_hidden_read_only.png)  