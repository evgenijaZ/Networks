# HTTP
1. Install and configure Apache server with PHP module.
2. Create virtual hosts.
3. Sing and install certificate, enable HTTPS.

| host name | DNS suffix    | port      | root folder    | access         |
|-----------|---------------|-----------|----------------|----------------|
| alpha     | zone01.com.ua |  80       | /web           |    all         |
| srv-01    | zone01.com.ua |  8000     | /usr/local/www | mercury, venus |

---
## Install Apache
```shell script
$ sudo apt-get update
$ sudo apt-get install apache2
```
## Configure virtual hosts  

Create folders
```shell script
$ sudo mkdir -p /web/alpha.zone01.com.ua/public_html
$ sudo mkdir -p /usr/local/www/srv-01.zone01.com.ua/public_html

$ sudo chown -R $USER:$USER /web/alpha.zone01.com.ua/public_html/
$ sudo chown -R $USER:$USER /usr/local/www/srv-01.zone01.com.ua/public_html/

$ sudo chmod -R 755 /usr/local/www/
$ sudo chmod -R 755 /web/
```

Add default index.html pages
```shell script
$ sudo nano /usr/local/www/srv-01.zone01.com.ua/public_html/index.html
```
```html
<html>
  <head>
    <title>Welcome to zone01.com.ua!</title>
  </head>
  <body>
    <h1>Success! The srv-01.zone01.com.ua virtual host is working!</h1>
  </body>
</html>
```
```shell script
$ sudo nano /web/alpha.zone01.com.ua/public_html/index.html
```
```html
<html>
  <head>
    <title>Welcome to zone01.com.ua!</title>
  </head>
  <body>
    <h1>Success! The alpha.zone01.com.ua virtual host is working!</h1>
  </body>
</html>
```
Configure `alpha.zone01.com.ua` host  
```shell script
$ sudo nano /etc/apache2/sites-available/alpha.zone01.com.ua.conf
```
```text
<VirtualHost *:80>

        ServerAdmin alpha@zone01.com.ua
        ServerName alpha.zone01.com.ua
        DocumentRoot /web/alpha.zone01.com.ua/public_html

        <Directory /web/alpha.zone01.com.ua/public_html>
           Options -Indexes +FollowSymLinks
           AllowOverride All
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```
Configure `srv-01.zone01.com.ua` host  
```shell script
$ sudo nano /etc/apache2/sites-available/srv-01.zone01.com.ua.conf
```
```text
<VirtualHost *:8000>

        ServerAdmin alpha@zone01.com.ua
        ServerName srv-01.zone01.com.ua
        DocumentRoot /usr/local/www/srv-01.zone01.com.ua/public_html/

         <Directory /usr/local/www/srv-01.zone01.com.ua/public_html/>
           Options -Indexes +FollowSymLinks
           AllowOverride All
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```
Remove default host configuration  
```shell script
$ sudo rm /etc/apache2/sites-available/000-default.conf  
```
Enable virtual hosts  
```shell script
$ sudo a2ensite alpha.zone01.com.ua.conf
$ sudo a2ensite srv-01.zone01.com.ua.conf
```
Configure ports
```shell script
$ sudo nano /etc/apache2/ports.conf
Listen 80
Listen 8000
```
```shell script
$ sudo service apache2 restart
```
Check if ports are enabled now
```shell script
$ sudo netstat -anp | grep apache
```

## Check configs
Check syntax
```shell script
$ sudo apachectl configtest 
```
Show all
```shell script
$ sudo apache2ctl -S  
```

## Configure hosts on Widows
Add the following lines into `C:\Windows\System32\drivers\etc\hosts`
```text
192.168.31.31 alpha.zone01.com.ua
192.168.31.31 srv-01.zone01.com.ua
```

 
## Setup certificates 
```shell script
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/alpha.zone01.com.ua.key -out /etc/ssl/certs/alpha.zone01.com.ua.crt
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/srv-01.zone01.com.ua.key -out /etc/ssl/certs/srv-01.zone01.com.ua.crt
```

```shell script
$ sudo nano /etc/apache2/sites-available/alpha.zone01.com.ua.conf
```
```text
SSLEngine on                                                                                            
SSLCertificateFile /etc/ssl/certs/alpha.zone01.com.ua.crt                                               
SSLCertificateKeyFile /etc/ssl/private/alpha.zone01.com.ua.key
```

```shell script
$ sudo nano /etc/apache2/sites-available/srv-01.zone01.com.ua.conf
```
```text
SSLEngine on                                                                                            
SSLCertificateFile /etc/ssl/certs/srv-01.zone01.com.ua.crt                                               
SSLCertificateKeyFile /etc/ssl/private/srv-01.zone01.com.ua.key
```

Enable SSL and restart server
```shell script
$ sudo a2enmod ssl
$ sudo service apache2 restart
```

> In browser, download certificate, install it and save to "Trusted Root Certification Authorities"


## Restrict access to `srv-01.zone01.com.ua` host
```shell script
$ sudo htpasswd -c /etc/apache2/.htpasswd mercury
$ sudo htpasswd /etc/apache2/.htpasswd venus
```

```shell script
$ sudo nano /etc/apache2/sites-available/srv-01.zone01.com.ua.conf
```
```text
  #<Directory /usr/local/www/srv-01.zone01.com.ua/public_html/>
	 #	...
	  AuthType Basic
      AuthName "Restricted Content"
      AuthUserFile /etc/apache2/.htpasswd
      Require valid-user
	 # ...
```
```shell script
$ sudo systemctl restart apache2
```

## Install and set up PHP
```shell script
$ sudo apt install php libapache2-mod-php
```
Make index.php default page
```shell script
$ sudo nano /etc/apache2/mods-enabled/dir.conf
```
Add to the beginning of pages list
```text
index.php
```
Add default `index.php` page
```shell script
$ sudo nano /web/alpha.zone01.com.ua/public_html/index.php
```
```text
<h1>PHP is working on alpha.zone01.com.ua</h1>
<?php 
    phpinfo();
?>
```


```shell script
$ sudo nano /usr/local/www/srv-01.zone01.com.ua/public_html/index.php
```
```text
<h1>PHP is working on srv-01.zone01.com.ua</h1>
<?php
    phpinfo();
?>
```

Check installed PHP version
```shell script
$ php -v
```

Enable php module based on installed version
```shell script
sudo a2enmod php7.4
```

---
## Results
Access to `srv-01.zone01.com.ua`  
![Access to `srv-01.zone01.com.ua`](/imgs/lab7_access.png)   
PHP module on `srv-01.zone01.com.ua`  
![PHP on `srv-01.zone01.com.ua`](/imgs/lab7_php.png)   
---
---  
[alpha.zone01.com.ua.conf](/resources/alpha.zone01.com.ua.conf)  
[srv-01.zone01.com.ua.conf](/resources/srv-01.zone01.com.ua.conf)  
