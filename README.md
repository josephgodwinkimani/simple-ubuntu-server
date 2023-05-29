# Ubuntu/Debian Server setup

How to setup an Ubuntu/Debian server with the following features:

- BindDNS Management
- MariaDB relational database
- Nginx Open Source webserver and reverse proxy
- Let's Encrypt

## Table of Contents

- [Configure Master BIND DNS Server](#configure-master-bind-dns-server)
- [Install and Configure PHP](#install-and-configure-php)
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
# aifilms.ai is the zone name
zone "aifilms.ai" IN { // Domain name

      type master; // Primary DNS

     file "/etc/bind/forward.aifilms.ai.db"; // Forward lookup file

     allow-update { none; }; // Since this is the primary DNS, it should be none.

};

zone "219.107.68.164.in-addr.arpa" IN { //Reverse lookup name, should match your network in reverse order

     type master; // Primary DNS

     file "/etc/bind/reverse.aifilms.ai.db"; //Reverse lookup file

     allow-update { none; }; //Since this is the primary DNS, it should be none.



};

```

3. Configure Bind DNS zone lookup files

Copy the sample forward zone lookup file to a file called `forward.aifilms.ai.db` under the /etc/bind directory

```bash
$ cp /etc/bind/db.local /etc/bind/forward.aifilms.ai.db
```

The acronyms on the file have the following description:

    SOA – Start of Authority
    NS – Name Server
    A – A record
    MX – Mail for Exchange
    CN – Canonical Name

We have to edit the zone file and update the content as below. Modify it as per your domain name:

```bash
$ nano /etc/bind/forward.aifilms.ai.db
```

```bash

$TTL    604800
@       IN      SOA     ns1.aifilms.ai. root.ns1.aifilms.ai. (
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

@        IN      NS      ns1.aifilms.ai.
@	 IN      A       194.163.166.170
@        IN      AAAA    2a02:c206:2120:3375:0000:0000:0000:0001

;IP address of Name Server

ns1     IN      A       194.163.166.170

;Mail Exchanger

aifilms.ai.   IN     MX   10   mail.aifilms.ai.

;A – Record HostName To Ip Address

www     IN       A      194.163.166.170
mail    IN       A      194.163.166.170

;CNAME record

ftp     IN      CNAME   www.aifilms.ai.

```

For the Reverse zone lookup file -

The acronyms in the revese zone file are:

    PTR – Pointer
    SOA – Start of Authority

Copy the sample reverse zone file in etc/bind to a file called `reverse.aifilms.ai.db`

```bash
$ cp /etc/bind/db.127 /etc/bind/reverse.aifilms.ai.db
$  /etc/bind/reverse.aifilms.ai.db

```

```bash

;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     aifilms.ai. root.aifilms.ai. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;

;Name Server Information

@       IN      NS     ns1.aifilms.ai.
ns1     IN      A       194.163.166.170
;Reverse lookup for Name Server

2      IN      PTR    ns1.aifilms.ai.

;PTR Record IP address to HostName

3     IN      PTR    www.aifilms.ai.
4     IN      PTR    mail.aifilms.ai.

```

4. Check BIND DNS syntax

The named-checkconf command is used to check if the syntax is okay or if there is any error. The command should return to shell if there is no error:

```bash
$ named-checkconf

#forward zone file
$ named-checkzone aifilms.ai /etc/bind/forward.aifilms.ai.db
#reverse zone file
$ named-checkzone 219.107.68.164.in-addr.arpa /etc/bind/reverse.aifilms.ai.db
# restart and enable BIND service
$ systemctl restart bind9
$ systemctl enable bind9
# Test DNS server
$ dig www.aifilms.ai
$ dig -x 194.163.166.170
```


### Install and Configure PHP

```bash
# install php 7.4 and PHP-FPM for processing the PHP files with Nginx
$ apt install php7.4 php7.4-common php7.4-mysql php7.4-xml php7.4-xmlrpc php7.4-curl php7.4-gd php7.4-imagick php7.4-cli php7.4-dev php7.4-imap php7.4-mbstring php7.4-opcache php7.4-soap php7.4-zip php7.4-intl php7.4-bcmath php7.4-json -y
$ systemctl start php7.4-fpm
$ systemctl enable php7.4-fpm
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
$ lsb_release -a
$ apt-get install apt-transport-https curl -y
$ curl -o /etc/apt/trusted.gpg.d/mariadb_release_signing_key.asc 'https://mariadb.org/mariadb_release_signing_key.asc'
$ sh -c "echo 'deb https://mariadb.mirror.liquidtelecom.com/repo/10.8/ubuntu focal main' >>/etc/apt/sources.list"
# Once the key is imported and the repository added you can install MariaDB 10.8 from the MariaDB repository with:
$ apt-get update
$ apt-get install mariadb-server -y
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
$ rsync -chavzP -e 'ssh -p 8123' ~/Projects/deployer/ root@194.163.166.170:/var/www/aifilms.ai
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

Create a Nginx configuration for your domain/website `nano /etc/nginx/sites-available/aifilms.ai`

```bash
server {
    listen 80;
    listen [::]:80;
    server_name aifilms.ai www.aifilms.ai;
    root /var/www/aifilms.ai;

    index index.html index.htm index.php;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /api/build/ { # resides in aifilms.ai/app/build/
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    location /app2/v1/ { # resides in aifilms.ai/app2/v1/
        proxy_pass http://localhost:3002/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # This configuration is open underaifilms.ai/api/v1
    # https://laravel.com/docs/9.x/deployment#nginx
    location /api/v1/public {
            # alias //var/www/aifilms.ai/api/v1/public;
            try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
     }

    location ~ /\.ht {
        deny all;
    }

}
```

Make sure you didn’t introduce any syntax errors by typing:

```bash
$ nginx -t
$ ln -d /etc/nginx/sites-available/aifilms.ai /etc/nginx/sites-enabled/
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
$ certbot --nginx -d aifilms.ai -d www.aifilms.ai
```

Edit the Nginx configuration for your domain/website `nano /etc/nginx/sites-available/aifilms.ai`

```bash
server {
    listen 80;
    listen [::]:80;
    server_name aifilms.ai www.aifilms.ai;
    root /var/www/aifilms.ai;

    index index.html index.htm index.php;

    listen 443 ssl; # managed by Certbot

    # RSA certificate
    ssl_certificate /etc/letsencrypt/live/aifilms.ai/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/aifilms.ai/privkey.pem; # managed by Certbot

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

    location /api/build/ { # resides in aifilms.ai/app/build/
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # This configuration is open underaifilms.ai/api/v1
    # https://laravel.com/docs/9.x/deployment#nginx
    location /api/v1/public {
            # alias //var/www/aifilms.ai/api/v1/public;
            try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
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

0 12 * * * /usr/bin/certbot renew --quiet --nginx # The command checks to see if the certificate on the server will expire within the next 30 days, and renews it if so

```

### Install a web GUI file manager

> TinyFileManager is web based file manager and it is a simple, fast and small file manager with a single file, multi-language ready web application for storing, uploading, editing and managing files and folders online via web browser. The Application runs on PHP 5.5+, It allows the creation of multiple users and each user can have its own directory and a build-in support for managing text files with cloud9 IDE and it supports syntax highlighting for over 150+ languages and over 35+ themes.

```bash
$ wget https://github.com/prasathmani/tinyfilemanager/archive/refs/tags/2.4.7.zip
$ unzip tinyfilemanager-2.4.7.zip
$ cd tinyfilemanager-2.4.7
# rename file to anything you need
$ mv -f /var/www/aifilms.ai/tinyfilemanager-2.4.7/tinyfilemanager.php /var/www/aifilms.ai/filemanager.php
$ nano tinyfilemanager.php
> $auth_users = array(
    'username' => 'REPLACE YOUR GENERATED PASSWORD HERE'
);
```
