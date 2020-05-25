# IMAP, POP3

1. Setup mail server with POP3 and IMAP  
2. Enable secure connection  

**Domain name:** zone01.com.ua  
**Connection type:** Plain, secure on the standard port (`STARTTLS`)
 
| User     | Auth type |
|----------|-----------|
| mercury  | CRAM-MD5  |
| venus    | Plain     |
| earth    | CRAM-MD5  |
| saturn   | Plain     |
| jupiter  | CRAM-MD5  |

---
### Install Dovecot  
```shell script
$ sudo apt-get install dovecot-imapd dovecot-pop3d
```

### Configure Dovecot  
```shell script
$ sudo nano  /etc/dovecot/dovecot.conf
```
Just ensure the following lines are present in the files or add them accordingly, do not delete the existing ones
```text
ssl = no
disable_plaintext_auth = no

protocols = pop3 imap

pop3_uidl_format = %g

auth_verbose = yes
auth_mechanisms = plain
passdb {
  driver = passwd-file
  args = /etc/dovecot/passwd
}
userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/vhost/%d/%n
}
```
Configure authentication  
```shell script
$ sudo nano /etc/dovecot/conf.d/10-auth.conf
```
```text
auth_mechanisms = plain login cram-md5
!include auth-passwdfile.conf.ext
```

Mail settings 
```shell script
$ sudo nano /etc/dovecot/conf.d/10-mail.conf
```
```text
mail_location = maildir:/home/vmail/%d/%n
mail_uid = 107
mail_gid = 115
first_valid_uid = 1
last_valid_uid = 10000

mail_privileged_group = mail
```
> There `first_valid_uid` and `last_valid_uid` could have other values to make narrower range, but it needs to be tested  
Enable services and specify user and group.  

> `%d` in `mail_location` stands for domain name, `%n` is user name. So, the mail for `mercury@zone01.com.ua` will be stored in `/home/vmail/zone01.com.ua/mercury` 
```shell script
$ sudo nano /etc/dovecot/conf.d/10-master.conf
```
```text
service imap-login {
  inet_listener imap {
    #port = 143
  }
  inet_listener imaps {
    #port = 993
    #ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    #port = 110
  }
  inet_listener pop3s {
    #port = 995
    #ssl = yes
  }
}

service auth {
unix_listener /var/spool/postfix/private/auth {
    mode = 0666
  }

  user = postfix
  group = postfix
}
```

### Configure Postfix  
```shell script
$ sudo nano /etc/postfix/master.cf
```
```text
dovecot   unix  -       n       n       -       -       pipe
  flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -f ${sender} -d ${user}@${nexthop}
```
```shell script
$ sudo nano /etc/postfix/main.cf
```
```text
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
virtual_transport = dovecot
dovecot_destination_recipient_limit = 1

queue_directory = /var/spool/postfix
```

### Manage permissions  
```shell script
$ sudo chown vmail:vmail /home/vmail/
$ chown -R :vmail /var/mail/vhost/
$ sudo chown postfix:postfix /var/run/dovecot/auth-worker
``` 

### Create users  
Generate passwords in CRAM-MD5
```shell script
$ doveadm pw -s CRAM-MD5 -u mercury@zone01.com.ua -p mercury
$ doveadm pw -s CRAM-MD5 -u earth@zone01.com.ua -p earth
$ doveadm pw -s CRAM-MD5 -u jupiter@zone01.com.ua -p jupiter
```

The results put in `/etc/dovecot/passwd`  
```shell script
$ sudo nano /etc/dovecot/passwd
```
```text
venus@zone01.com.ua:{PLAIN}venus::::::
saturn@zone01.com.ua:{PLAIN}saturn::::::
mercury@zone01.com.ua:{CRAM-MD5}3a7fef2e7049c2110d6e331acbd64aa516a20a4e7e90bbb9a01927f895444702:postfix:postfix::
earth@zone01.com.ua:{CRAM-MD5}4586faf6c8071a15b0e3e239ef7fa32ba45326f4b4d26d1dc6a9d7ce7c5f490e:postfix:postfix::
jupiter@zone01.com.ua:{CRAM-MD5}24cefdb114cbb56e7da1b0fedb4eca4b45b8838f7b2f7e59cce4fe21b4844169:postfix:postfix::
```
Test access  
```shell script
$ sudo doveadm auth test -x service=imap -x rip=mail.zone01.com.ua mercury@zone01.com.ua mercury  
```
> Where `mercury@zone01.com.ua` is user , `mercury` is pass  

Add Dovecot users to Postfix to allow use them as recipients
```shell script
$ sudo nano /etc/postfix/vmailbox
```
```text
mercury@zone01.com.ua   zone01.com.ua/mercury
venus@zone01.com.ua     zone01.com.ua/venus
earth@zone01.com.ua     zone01.com.ua/earth
saturn@zone01.com.ua    zone01.com.ua/saturn
jupiter@zone01.com.ua   zone01.com.ua/jupiter
```

Apply changes and restart services  
```shell script
$ sudo postmap /etc/postfix/vmailbox
$ sudo postfix reload
$ sudo /etc/init.d/dovecot restart
```
Connect via SSL

POP3  
`openssl s_client -connect mail.zone01.com.ua:pop3s` or `openssl s_client -connect mail.zone01.com.ua:pop3 -starttls pop3`  
IMAP  
`openssl s_client -connect mail.zone01.com.ua:imaps` or `openssl s_client -connect mail.zone01.com.ua:imap -starttls imap`

### Test CRAM-MD5 authentication  

CRAM-MD5 password is not transmitted as plain text, hash comparison is used instead  

```shell script
$ sudo nano md5cram.pl
```
```text
use strict;
use MIME::Base64 qw(encode_base64 decode_base64);
use Digest::HMAC_MD5;

die "Usage: $0 username password ticket\n" unless $#ARGV == 2;

my ($username, $password, $ticket64) = @ARGV;

my $ticket = decode_base64($ticket64) or
die ("Unable to decode Base64 encoded string '$ticket64'\n");
my $password_md5 = Digest::HMAC_MD5::hmac_md5_hex($ticket, $password);
print encode_base64 ("$username $password_md5", "");
```
Run script with login, password and keyword received when trying to auth using cram-md5 to retrieve hash.  
Example:
```shell script
$ sudo perl md5cram.pl mercury@zone01.com.ua mercury PDg4NDMwODY1MjQwMDk3ODYuMTU5MDQ0NjI0NUB1YnVudHVfc2VydmVyPg==
```
> `mercury@zone01.com.ua` is login, `mercury` is password, 
`PDg4NDMwODY1MjQwMDk3ODYuMTU5MDQ0NjI0NUB1YnVudHVfc2VydmVyPg==` is keyword received from the server as response on `01 AUTHENTICATE  CRAM-MD5` (IMAP) or `AUTH CRAM-MD5` (POP3)

### Mails
Mails should be available in the `/home/vmail/%domain%/%user%/new`  

---  
### Results
POP3  
![POP3](/imgs/lab3_POP3.png)  
IMAP  
![IMAP](/imgs/lab3_IMAP.png)  
Authentication using CRAM-MD5  
![CRAM-MD5](/imgs/lab3_CRAM-MD5.png)  

