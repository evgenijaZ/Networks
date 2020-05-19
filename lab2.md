# SMTP  
1. Setup mail server  
2. Allow to send mail only authenticated users except of allowed IPs  
3. Enable black lists  
4. Enable aliases  
5. Enable SSL/TLS

**Zone:** zone01.com.ua  
**Allowed IP:** 192.168.31.101  

| User                 | Password  | Alias |
|----------------------|-----------|-------|
| alpha@zone01.com.ua  | alpha     | ws1   |
| beta@zone01.com.ua   | beta      | ws2   |
| gamma@zone01.com.ua  | gamma     | ws3   |
| delta@zone01.com.ua  | delta     | ws4   |
| omega@zone01.com.ua  | omega     | ws5   |  

---  
### Add MX record
```shell script
sudo nano /etc/bind/db.zone01
```
```text
          IN      MX  10  mail.zone01.com.ua.
mail      IN      A       192.168.31.31
```
Check from client
```shell script
dig zone01.com.ua MX
```

### Install Postfix
```shell script
sudo apt-get install postfix
```

### Configure mail server
```shell script
sudo nano /etc/postfix/main.cf
```
```text
myhostname = mail.zone01.com.ua
mydomain = zone01.com.ua
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = localhost
relayhost =
#mynetworks = localhost
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

inet_protocols = all

mailbox_transport = lmtp:unix:/var/run/cyrus/socket/lmtp
local_recipient_maps =
```

### SASL  
```shell script
sudo apt-get install sasl2-bin
sudo nano /etc/default/saslauthd
```
```text
START=yes
MECHANISMS="sasldb"
```

```shell script
sudo nano /etc/postfix/sasl/smtpd.conf
```
```text
pwcheck_method: saslauthd
```

```shell script
sudo nano /etc/postfix/main.cf
```
```text
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
broken_sasl_auth_clients = yes
smtpd_sasl_local_domain =
smtpd_sasl_authenticated_header = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
```

```shell script
sudo rm -r /var/run/saslauthd/
sudo mkdir -p /var/spool/postfix/var/run/saslauthd
sudo ln -s /var/spool/postfix/var/run/saslauthd /var/run
sudo chgrp sasl /var/spool/postfix/var/run/saslauthd
sudo adduser postfix sasl
```
Restart Postfix  
```shell script
sudo /etc/init.d/postfix restart
sudo /etc/init.d/saslauthd start
```

### Add users  
```shell script
sudo saslpasswd2 -c -u zone01.com.ua alpha
sudo sasldblistusers2
```
> where `alpha` - user name, `zone01.com.ua` - domain name

Encode text to Base64
```shell script
perl -MMIME::Base64 -e "print encode_base64('alpha@zone01.com.ua');"
```

### Aliases  
```shell script
sudo touch /etc/postfix/aliases
sudo newaliases
```
```shell script
sudo nano /etc/postfix/vmailbox
```
```text
alpha@zone01.com.ua     zone01.com.ua/alpha
beta@zone01.com.ua      zone01.com.ua/beta
gamma@zone01.com.ua     zone01.com.ua/gamma
delta@zone01.com.ua     zone01.com.ua/delta
omega@zone01.com.ua     zone01.com.ua/omega
```

```shell script
sudo nano /etc/postfix/valias
```
```text
ws1@zone01.com.ua       alpha@zone01.com.ua
ws2@zone01.com.ua       beta@zone01.com.ua
ws3@zone01.com.ua       gamma@zone01.com.ua
ws4@zone01.com.ua       delta@zone01.com.ua
ws5@zone01.com.ua       omega@zone01.com.ua
```

Run
`id postfix`  
Result put in `main.cf` in `virtual_uid_maps, virtual_gid_maps`

```shell script
sudo nano /etc/postfix/main.cf
```
```text
virtual_mailbox_base = /var/mail/vhost
virtual_mailbox_maps = hash:/etc/postfix/vmailbox
virtual_minimum_uid = 100
virtual_uid_maps = static:113
virtual_gid_maps = static:113
virtual_alias_maps = hash:/etc/postfix/valias
virtual_mailbox_domains = $mydomain
```

Apply aliases
```shell script
sudo postmap /etc/postfix/valias
sudo postmap /etc/postfix/vmailbox
sudo postfix reload
```

### Certificates  
```shell script
sudo openssl req -new -x509 -nodes -out smtpd.pem -keyout smtpd.pem -days 3650
```

### TLS
```shell script
sudo nano /etc/postfix/main.cf
```
```text
smtp_use_tls = yes
smtpd_use_tls = yes
smtp_tls_note_starttls_offer = yes
smtpd_tls_key_file = /etc/postfix/ssl/smtpd.pem
smtpd_tls_cert_file = $smtpd_tls_key_file
smtpd_tls_CAfile = $smtpd_tls_key_file
smtpd_tls_loglevel = 5
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s
tls_random_source = dev:/dev/urandom

smtp_tls_security_level = may
```

### White/Black list  
Configure recipients black list
```shell script
sudo nano /etc/postfix/main.cf
```
```text
smtpd_recipient_restrictions =
   check_client_access hash:/etc/postfix/client_checks,
   check_sender_access hash:/etc/postfix/sender_checks
```
```shell script
sudo nano /etc/postfix/client_checks
```
```text
example.com               REJECT
example1.com              OK
```
```shell script
sudo nano /etc/postfix/sender_checks
```
```text
gamma@zone01.com.ua         REJECT
example2.com                OK
```
```shell script
postmap /etc/postfix/client_checks
postmap /etc/postfix/sender_checks
/etc/init.d/postfix reload
```
### Allow only logged in  
```shell script
sudo nano /etc/postfix/master.cf
```
```text
-o smtpd_relay_restrictions=permit_sasl_authenticated,permit_mynetworks,reject
```
```shell script
sudo nano /etc/postfix/main.cf
```
```text
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.31.101 192.168.31.31
```
> where `192.168.31.101 192.168.31.31` added as IP allowed without authentication  

### Check mail list  
```shell script
sudo nano /var/mail/vhost/zone01.com.ua/alpha 
```
> where `alpha` - username, `zone01.com.ua` - zone domain

### Check all configuration  
```shell script
postconf -n
```

### SSL
```shell script
sudo nano /etc/postfix/master.cf
```
```text
465     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
```
```shell script
sudo postfix restart
```
Check if running on 465 port  
```shell script
sudo netstat -tlanp | grep 465
```
Connect via STARTTLS  
```shell script
openssl s_client -connect 192.168.31.31:25 -starttls smtp  
```
>Type all commands in lower case (in upper case could be "554 5.5.1 Error: no valid recipients")

### Session example
```shell script
ubuntu@ubuntu_server:~$ telnet localhost 25
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
220 mail.zone01.com.ua ESMTP Postfix (Ubuntu)
mail from: alpha@zome01.com.ua
250 2.1.0 Ok
rcpt to: beta@zone01.com.ua
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
hello from alpha to mercury
.
250 2.0.0 Ok: queued as D21EC2152C
quit
221 2.0.0 Bye
Connection closed by foreign host.
```
---  
## Results  
Black list for sender  
![Black list for sender](/imgs/lab2_black_list.png)   
Required authentication from unknown host  
![Required authentication from unknown host](/imgs/lab2_unknown_host.png)  
Unknown host vs allowed IP  
![Unknown host vs allowed IP](/imgs/lab2_unknown_host_and_allowed.png)  
SSL/TLS  
![SSL/TLS](/imgs/lab2_ssl_tls.png)  

---  
[main.cf](/resources/main.cf)
