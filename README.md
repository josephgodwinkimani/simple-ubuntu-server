# Ubuntu Server setup

How to setup an Ubuntu server with the following features:

- BindDNS Management
- MariaDB relational database
- Nginx Open Source webserver and reverse proxy
- Let's Encrypt

## Table of Contents

- [Configure Master BIND DNS Server](#configure-master-bind-dns-server)
- [Install and Configure Nginx](#install-and-configure-nginx)
- [Install and Configure MariaDB](#install-and-configure-mariadb)
- [Setup your first NodeJS app](#setup-your-first-nodejs-app)
- [Install and Configure the Lets Encrypt Client](#install-and-configure-the-lets-encrypt-client)
- [Install a web GUI file manager](#install-a-web-gui-file-manager)


### Install and Configure Config Server Firewall

[CSF](https://download.configserver.com/csf/install.txt) checks the logs for failed login attempts at regular time interval, and is able to recognize most unauthorized attempts to gain access to your cloud server. You can define the desired action CSF takes and after how many attempts in the configuration file.

The following applications are supported by this feature:

    Courier imap, Dovecot, uw-imap, Kerio
    openSSH
    cPanel, WHM, Webmail (cPanel servers only)
    Pure-ftpd, vsftpd, Proftpd
    Password protected web pages (htpasswd)
    Mod_security failures (v1 and v2)
    Suhosin failures
    Exim SMTP AUTH

Services using the open ports:

    Port 20: FTP data transfer
    Port 21: FTP control
    Port 22: Secure shell (SSH) => this should always be changed to 8123 etc
    Port 25: Simple mail transfer protocol (SMTP)
    Port 53: Domain name system (DNS)
    Port 80: Hypertext transfer protocol (HTTP)
    Port 110: Post office protocol v3 (POP3)
    Port 113: Authentication service/identification protocol
    Port 123: Network time protocol (NTP)
    Port 143: Internet message access protocol (IMAP)
    Port 443: Hypertext transfer protocol over SSL/TLS (HTTPS)
    Port 465: URL Rendesvous Directory for SSM (Cisco)
    Port 587: E-mail message submission (SMTP)
    Port 993: Internet message access protocol over SSL (IMAPS)
    Port 995: Post office protocol 3 over TLS/SSL (POP3S)

```bash
$ wget http://download.configserver.com/csf.tgz
$ tar -xzf csf.tgz
$ ufw disable
$ cd csf && sh install.sh
# check if the required iptables modules are available
$ perl /usr/local/csf/bin/csftest.pl
# change setting TESTING at the beginning of the configuration file to 0 i.e TESTING = "0"
$ nano /etc/csf/csf.conf
$ csf -r
# Allowing IP addresses
$ nano /etc/csf/csf.allow
# Ignoring IP addresses
$ nano /etc/csf/csf.ignore
# change ssh port
$ nano /etc/ssh/sshd_config
$ service ssh restart
```

### Configure Master BIND DNS Server

> BIND is a suite of software for interacting with the Domain Name System. Its most prominent component, named, performs both of the main DNS server roles, acting as an authoritative name server for DNS zones and as a recursive resolver in the network.

1. Install the necessary packages

```bash
$ apt install -y bind9 bind9utils bind9-doc dnsutils
```

2. Create the forward and reverse zones in global conf `nano /etc/bind/named.conf.local`

```bash
# mbuddyx.com is the zone name
zone "mbuddyx.com" IN { // Domain name

      type master; // Primary DNS

     file "/etc/bind/forward.mbuddyx.com.db"; // Forward lookup file

     allow-update { none; }; // Since this is the primary DNS, it should be none.

};

zone "219.107.68.164.in-addr.arpa" IN { //Reverse lookup name, should match your network in reverse order

     type master; // Primary DNS

     file "/etc/bind/reverse.mbuddyx.com.db"; //Reverse lookup file

     allow-update { none; }; //Since this is the primary DNS, it should be none.



};

```

3. Configure Bind DNS zone lookup files

Copy the sample forward zone lookup file to a file called `forward.mbuddyx.com.db` under the /etc/bind directory

```bash
$ cp /etc/bind/db.local /etc/bind/forward.mbuddyx.com.db
```

The acronyms on the file have the following description:

    SOA – Start of Authority
    NS – Name Server
    A – A record
    MX – Mail for Exchange
    CN – Canonical Name

We have to edit the zone file and update the content as below. Modify it as per your domain name:

```bash
$ nano /etc/bind/forward.mbuddyx.com.db
```

```bash

$TTL    604800
@       IN      SOA     ns1.mbuddyx.com. root.ns1.mbuddyx.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
;@      IN      NS      localhost.
;@      IN      A       127.0.0.1
;@      IN      AAAA    ::1

;Name Server Information

@        IN      NS      ns1.mbuddyx.com.
@	 IN      A       164.68.107.219
@        IN      AAAA    2a02:c207:2097:1942:0000:0000:0000:0001

;IP address of Name Server

ns1     IN      A       164.68.107.219

;Mail Exchanger

mbuddyx.com.   IN     MX   10   mail.mbuddyx.com.

;A – Record HostName To Ip Address

www     IN       A      164.68.107.219
mail    IN       A      164.68.107.219

;CNAME record

ftp     IN      CNAME   www.mbuddyx.com.

```

For the Reverse zone lookup file -

The acronyms in the revese zone file are:

    PTR – Pointer
    SOA – Start of Authority

Copy the sample reverse zone file in etc/bind to a file called `reverse.mbuddyx.com.db`

```bash
$ cp /etc/bind/db.127 /etc/bind/reverse.mbuddyx.com.db

```

```bash

;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     mbuddyx.com. root.mbuddyx.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;

;Name Server Information

@       IN      NS     ns1.mbuddyx.com.
ns1     IN      A       164.68.107.219
;Reverse lookup for Name Server

2      IN      PTR    ns1.mbuddyx.com.

;PTR Record IP address to HostName

3     IN      PTR    www.mbuddyx.com.
4     IN      PTR    mail.mbuddyx.com.

```

4. Check BIND DNS syntax

The named-checkconf command is used to check if the syntax is okay or if there is any error. The command should return to shell if there is no error:

```bash
$ named-checkconf

#forward zone file
$ named-checkzone mbuddyx.com /etc/bind/forward.mbuddyx.com.db
#reverse zone file
$ named-checkzone 219.107.68.164.in-addr.arpa /etc/bind/reverse.mbuddyx.com.db
# restart and enable BIND service
$ systemctl restart bind9
$ systemctl enable bind9
# Test DNS server
$ /etc/resolv.conf
$ dig www.mbuddyx.com
$ dig -x 164.68.107.219
```

### Install and Configure Nginx

```bash
$ apt-get install nginx
# Enable the service
$ systemctl enable nginx
$ systemctl status nginx
```

### Install and Configure MariaDB

1. Installation

```bash
$ apt-get install apt-transport-https curl
$ curl -o /etc/apt/trusted.gpg.d/mariadb_release_signing_key.asc 'https://mariadb.org/mariadb_release_signing_key.asc'
$ sh -c "echo 'deb https://mariadb.mirror.liquidtelecom.com/repo/10.8/ubuntu focal main' >>/etc/apt/sources.list"
# Once the key is imported and the repository added you can install MariaDB 10.8 from the MariaDB repository with:
$ apt-get update
$ apt-get install mariadb-server
```

2. Configuring MariaDB

```bash
$ mysql_secure_installation
# Adjust the root password
$ mysql -u root
> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';
> GRANT ALL ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
> FLUSH PRIVILEGES;
> EXIT;
# Testing
$ systemctl status mariadb
$ mysqladmin version
$ mysqladmin -u root -p version
```

### Setup your first NodeJS app

```bash
# Install the latest LTS release of Node.js, using the NodeSource package archives - https://github.com/nodesource/distributions
$ curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
$ sudo apt-get install -y nodejs
$ node -v
$ npm -v
# Install Yarn v1
$ npm install yarn -g
# Deploy your NodeJS app
$ rsync -chavzP -e 'ssh -p 8123' ~/Projects/deployer/ root@164.68.107.219:/var/www/mbuddyx.com
# Install PM2 process manager that makes it possible to daemonize applications so that they will run in the background as a service
$ npm install pm2@latest -g
$ pm2 start build/app.js
$ pm2 startup systemd
$ pm2 save
$ systemctl start pm2
$ systemctl status pm2
# List the applications currently managed by PM2
$ pm2 list
# Display the application status, CPU, and memory usage
$ pm2 monit
```

### Setting Up Nginx as a Reverse Proxy Server

Create a Nginx configuration for your domain/website `nano /etc/nginx/sites-available/mbuddyx.com`

```bash
server {
    listen 80;
    server_name mbuddyx.com www.mbuddyx.com;
    root /var/www/mbuddyx.com;

    index index.html index.htm index.php;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /app/ { # resides in mbuddyx.com/app/
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /app2/v1/ { # resides in mbuddyx.com/app2/v1/
        proxy_pass http://localhost:3002/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
     }

    location ~ /\.ht {
        deny all;
    }

}
```

Make sure you didn’t introduce any syntax errors by typing:

```bash
$ nginx -t
$ systemctl restart nginx
```

### Install and Configure the Lets Encrypt Client

```bash
$ sudo apt-get install certbot
$ apt-get install python3-certbot-nginx
$ nginx -t && nginx -s reload
```

**Obtain the SSL/TLS Certificate**

```bash
$ certbot --nginx -d mbuddyx.com -d www.mbuddyx.com
```

Edit the Nginx configuration for your domain/website `nano /etc/nginx/sites-available/mbuddyx.com`

```bash
server {
    listen 80;
    server_name mbuddyx.com www.mbuddyx.com;
    root /var/www/mbuddyx.com;

    index index.html index.htm index.php;

    listen 443 ssl; # managed by Certbot

    # RSA certificate
    ssl_certificate /etc/letsencrypt/live/mbuddyx.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/mbuddyx.com/privkey.pem; # managed by Certbot

    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot

    # Redirect non-https traffic to https
    if ($scheme != "https") {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /app/ {
        proxy_pass http://localhost:3001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
     }

    location ~ /\.ht {
        deny all;
    }

}
```

**Delete the SSL Certificate**

Get the certificate's name that will delete:

```bash
$ certbot certificates
```

Delete only one certificate by the name

```bash
$ certbot delete --cert-name server.domain.tld
```

**Automatically Renew Let’s Encrypt Certificates**

Open the crontab file:

```bash
$ crontab -e

> 0 12 * * * /usr/bin/certbot renew --quiet --nginx # The command checks to see if the certificate on the server will expire within the next 30 days, and renews it if so

```

### Install a web GUI file manager

> TinyFileManager is web based file manager and it is a simple, fast and small file manager with a single file, multi-language ready web application for storing, uploading, editing and managing files and folders online via web browser. The Application runs on PHP 5.5+, It allows the creation of multiple users and each user can have its own directory and a build-in support for managing text files with cloud9 IDE and it supports syntax highlighting for over 150+ languages and over 35+ themes.

```bash
# install php 7.4
$ apt install php7.4-fpm php7.4-common php7.4-mysql php7.4-xml php7.4-xmlrpc php7.4-curl php7.4-gd php7.4-imagick php7.4-cli php7.4-dev php7.4-imap php7.4-mbstring php7.4-opcache php7.4-soap php7.4-zip php7.4-intl php7.4-bcmath
$ wget https://github.com/prasathmani/tinyfilemanager/archive/refs/tags/2.4.7.zip
$ unzip tinyfilemanager-2.4.7.zip
$ cd tinyfilemanager-2.4.7
# rename file to anything you need
$ mv -f /var/www/mbuddyx.com/tinyfilemanager-2.4.7/tinyfilemanager.php /var/www/mbuddyx.com/filemanager.php
$ nano tinyfilemanager.php
> $auth_users = array(
    'username' => 'REPLACE YOUR GENERATED PASSWORD HERE'
);
```
