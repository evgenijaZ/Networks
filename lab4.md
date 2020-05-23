# FTP  
1. Setup FTP server  
2. Allow access for anonymous user  
3. Create folder `/pub` with read-only and `/incoming` with read-write permissions for anonymous user  
4. Allow access for non-anonymous users `mercury` and `earth`  
5. `mercury` should have access only to `mercury`'s folder, `earth` - to whole file tree (working FTP root folder)  
---  
### Install VSFTPD
```shell script
$ sudo apt-get install vsftpd
```
### Configure FTP server
```shell script
$ sudo nano /etc/vsftpd.conf
```
```text
listen=NO
listen_ipv6=YES

#anonymous user
anonymous_enable=YES
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
no_anon_password=YES
anon_root=/var/ftp/ftp
chown_uploads=YES
chown_username=ftp
anon_world_readable_only=NO

local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pasv_enable=Yes
pasv_min_port=10000
pasv_max_port=11000
user_sub_token=$USER

#users' root folders
local_root=/var/ftp/$USER

userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO

# certificate and ssl
rsa_cert_file=/etc/cert/vsftpd.pem
rsa_private_key_file=/etc/cert/vsftpd.pem
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH

# to deal with '530 Non-anonymous sessions must use encryption'
force_local_logins_ssl=NO
force_local_data_ssl=NO

# to deal with '530 Login incorrect'
pam_service_name=ftp
```
Start the service
```shell script
$ systemctl start vsftpd
$ systemctl enable vsftpd
```
### Add users and setup permissions  

Add anonymous user for FTP
```shell script
$ adduser ftp
```

Create directory for the user in /home and within working FTP folder
```shell script
$ mkdir /home/ftp
$ mkdir /var/ftp/ftp
```

Create read-only folder for anonymous user `ftp`
```shell script
$ mkdir /var/ftp/ftp/pub
$ chmod 755 /var/ftp/ftp/pub/
```

Create a folder with read-write permissions
```shell script
$ chmod 777 /var/ftp/ftp/incoming/
$ mkdir /var/ftp/ftp/incoming
```

Add non-anonymous user `mercury` and create folders
```shell script
$ adduser mercury
$ mkdir /home/mercury
$ mkdir /var/ftp/mercury
```
Add non-anonymous user `earth` and create folders
```shell script
$ adduser earth
$ mkdir /home/earth
$ mkdir /var/ftp/earth
```
Add users to user list (to allow FTP usage)
```shell script
$ sudo nano /etc/vsftpd.userlist
```
```text
ftp
mercury
earth
```
Configure root folder for `earth` and write permissions
```shell script
$ sudo nano /etc/vsftpd/user_config_dir/earth
```
```text
local_root=/var/ftp
write_enable=YES
```
Change FTP working folder owners
```shell script
$ chown nobody:nogroup /var/ftp
```

### Create a certificate  
```shell script
$ mkdir /etc/cert
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/cert/vsftpd.pem -out /etc/cert/vsftpd.pem
```

Restart FTP server
```shell script
$ service vsftpd restart
```

---
### Results  
Anonymous user can only read in `/pub` folder, can read, write and delete in `/incoming` folder  
![Anonymous](/imgs/lab4_anonymous.png)  
`mercury` has access to his own folder, `earth` has access to whole file tree  
![Different access levels](/imgs/lab4_earth_mercury.png)  
