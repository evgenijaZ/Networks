# DNS
1. Setup Forward and Reverse Lookup zones in DNS  
2. Setup reserve DNS server  

**Zone:** zone01.com.ua  
**Main server:** 192.168.31.31  
**Reserve server:** 192.168.31.101
___  
### Install Bind  
```shell script
$ sudo apt-get install bind9
```  

Go to Bind folder  
```shell script
$ cd /etc/bind/
```

### Configure zones file
```shell script
$ sudo touch named.conf.my-zones
$ sudo chown bind:bind named.conf.my-zones
$ sudo nano named.conf.my-zones
```

```text
zone "zone01.com.ua." {
        type master;
        file "/etc/bind/db.zone01";
	allow-transfer { 192.168.31.101; };
};

zone "192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.192";
        allow-transfer { 192.168.31.101; };
};
```
> `allow-transfer { 192.168.31.101; };` is needed for reserve server  
 
### Forward lookup zone  
```shell script
$ sudo touch db.zone01
$ sudo chown bind:bind db.zone01
$ sudo nano db.zone01
```
```text
$TTL    3600
@ IN  SOA  zone01.com.ua. root.zone01.com.ua. (
                         2020020421        ; Serial
                         10800             ; Refresh
                         900               ; Retry
                         604800            ; Expire
                         86400 )           ; Negative Cache TTL

@         IN      NS      ns.zone01.com.ua.
ns        IN      A       192.168.31.31
alpha     IN      A       192.168.31.101
beta      IN      A       192.168.31.20
gamma     IN      A       192.168.31.30
delta     IN      A       192.168.31.40
omega     IN      A       192.168.31.50

ws1       IN      CNAME   alpha.zone01.com.ua.
ws2       IN      CNAME   beta.zone01.com.ua.
ws3       IN      CNAME   gamma.zone01.com.ua.
ws4       IN      CNAME   delta.zone01.com.ua.
ws5       IN      CNAME   omega.zone01.com.ua.
```
> where `192.168.31.31` - localhost IP, `192.168.31.20-50,101` - root IPs  

### Reverse lookup zone  
```shell script
$ sudo touch db.192
$ sudo chown bind:bind db.192
$ sudo nano db.192
```
```text
TTL    3600
@ IN  SOA zone01.com.ua. root.zone01.com.ua. (
                         2020020421        ; Serial
                         10800             ; Refresh
                         900               ; Retry
                         604800            ; Expire
                         86400 )           ; Negative Cache TTL
;
@      IN      NS        ns.zone01.com.ua.

10     IN      PTR       alpha.zone01.com.ua.
20     IN      PTR       beta.zone01.com.ua.
30     IN      PTR       gamma.zone01.com.ua.
40     IN      PTR       delta.zone01.com.ua.
50     IN      PTR       omega.zone01.com.ua.
```

```shell script
$ sudo nano named.conf
```
```text
include "/etc/bind/named.conf.my-zones";
```
```shell script
$ sudo nano named.conf.options
```
```text
options {
        directory "/var/cache/bind";

        forwarders {
                192.168.31.1;
        };
        listen-on port 53 {
                localhost; 192.168.31.0/24;
        };
        allow-query {
                localhost; 192.168.31.0/24;
        };
        recursion yes;

        dnssec-validation no;
```

### Restart Bind  
```shell script
$ /etc/init.d/bind9 restart
$ sudo systemctl enable bind9
$ sudo systemctl restart bind9
```
### Add nameserver
```shell script
$ sudo apt install resolvconf
$ sudo nano /etc/resolvconf/resolv.conf.d/head
```
```text
nameserver 127.0.0.1
```
### Tests  
Check zones syntax
```shell script
$ named-checkconf
$ named-checkzone zone01.com.ua /etc/bind/db.zone01
```
Check using dig
```shell script
$ dig +short @192.168.31.101 A ws1.zone01.com.ua
```
Check using nslookup
```shell script
$ nslookup alpha.zone01.com.ua
```
---  

## Slave DNS server  
Go to Slave server node
### Install Bind  
```shell script
$ sudo apt-get install bind9
```  

Go to Bind folder  
```shell script
$ cd /etc/bind/
```

### Configure server  
```shell script
$ sudo touch named.conf.my-zones
$ sudo chown bind:bind named.conf.my-zones
$ sudo nano named.conf.my-zones
```

```text
zone "zone01.com.ua." {
        type slave;
        file "/etc/bind/db.zone01";
	masters { 192.168.31.31; };
};

zone "172.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.192";
        masters { 192.168.31.31; };
};
```

### Forward lookup zone  
```shell script
$ sudo touch db.zone01
$ sudo chown bind:bind db.zone01
$ sudo nano db.zone01
```
```text
$TTL    3600
@ IN  SOA  zone01.com.ua. root.zone01.com.ua. (
                         2020020421        ; Serial
                         10800             ; Refresh
                         900               ; Retry
                         604800            ; Expire
                         86400 )           ; Negative Cache TTL

@         IN      NS      ns.zone01.com.ua.
ns        IN      A       192.168.31.31
alpha     IN      A       192.168.31.101
beta      IN      A       192.168.31.20
gamma     IN      A       192.168.31.30
delta     IN      A       192.168.31.40
omega     IN      A       192.168.31.50

ws1       IN      CNAME   alpha.zone01.com.ua.
ws2       IN      CNAME   beta.zone01.com.ua.
ws3       IN      CNAME   gamma.zone01.com.ua.
ws4       IN      CNAME   delta.zone01.com.ua.
ws5       IN      CNAME   omega.zone01.com.ua.
```

### Reverse lookup zone  
```shell script
$ sudo touch db.192
$ sudo chown bind:bind db.192
$ sudo nano db.192
```
```text
TTL    3600
@ IN  SOA zone01.com.ua. root.zone01.com.ua. (
                         2020020421        ; Serial
                         10800             ; Refresh
                         900               ; Retry
                         604800            ; Expire
                         86400 )           ; Negative Cache TTL
;
@      IN      NS        ns.zone01.com.ua.

10     IN      PTR       alpha.zone01.com.ua.
20     IN      PTR       beta.zone01.com.ua.
30     IN      PTR       gamma.zone01.com.ua.
40     IN      PTR       delta.zone01.com.ua.
50     IN      PTR       omega.zone01.com.ua.
```

```shell script
$ sudo nano named.conf
```
```text
include "/etc/bind/named.conf.my-zones";
```
```shell script
$ sudo nano named.conf.options
```
```text
options {
        directory "/var/cache/bind";
	
	dnssec-validation auto;

        auth-nxdomain no;    # conform to RFC1035
        allow-query { any; };
        allow-transfer { none; };
        allow-update { 192.168.31.31; };
        recursion yes;
        allow-recursion { 127.0.0.1; };
        notify yes;

        listen-on-v6 { any; };

};
```

---  
### Issues
> dumping master file: /etc/bind/tmp-W7fCh814Kz: open: permission denied  

```shell script
$ sudo nano /etc/apparmor.d/usr.sbin.named
```
```text
/etc/bind/** rw,
```
 
```shell script
$ service apparmor restart
$ service bind9 restart
```
## Results  
Master server  
![Master server example](/imgs/lab1.png)  
Slave server  
![Slave server example](/imgs/lab1-slave.png)
