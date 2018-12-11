# Setup Drupal and CiViCRM 

This document is written base on:
- CentOS 7 (7.6)
- PHP (PHP 5.6.39)
- PHP-FPM (PHP 5.6.39)
- MySQL (5.7.24 MySQL Community Server)
- Drupal 7
- CiViCRM (4.6.34)
- Nginx (nginx/1.12.2)

**Here is follow of our site:**

```
users --> https://tasks.rivieu.com/ --> Nginx --> PHP-FPM --> Drupal/CiviCRM --> MySQL
```

**Here is the documents that I have reference:**
- Prepare environment and dependecies: 
https://www.hostinger.com/tutorials/how-to-install-lemp-centos7#gref
https://www.rosehosting.com/blog/how-to-install-drupal-7-on-centos-7-with-nginx-mariadb-and-php-fpm/
- Create https by use Let's Encrypt: https://www.nginx.com/blog/free-certificates-lets-encrypt-and-nginx/
- Install CiViCRM with Drupal: https://docs.civicrm.org/sysadmin/en/latest/install/drupal7/

# How to install

## Prepare
- Make sure you have a root permission (or sudo permission) on the server which you will install Drupal/CiViCRM.
- Make sure you have a public domain map to public ip and the DNS A/AAAA record(s) for that domain
   contain(s) the right public IP address.

## Nginx
- Install EPEL repository by running this command:
```sudo yum install epel-release```
- Update OS and packages
```sudo yum update```
- Install Nginx
```sudo yum install nginx```

## PHP
- Install additional CentOS repo which contains required packages for PHP v5.6:
```sudo rpm -Uvh http://rpms.remirepo.net/enterprise/remi-release-7.rpm```
- Enable php56 repository
```sudo yum install yum-utils```
```sudo yum-config-manager --enable remi-php56```
- Install PHP package
```sudo yum --enablerepo=remi,remi-php56 install php-fpm php-common```
- Install common modules
```sudo yum --enablerepo=remi,remi-php56 install php-opcache php-pecl-apcu php-cli php-pear php-pdo php-mysqlnd php-pgsql php-pecl-mongodb php-pecl-redis php-pecl-memcache php-pecl-memcached php-gd php-mbstring php-mcrypt php-xml```
- Install Drush
```sudo yum install drush```

## MySQL
- Install mysql repository
```sudo yum localinstall https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm```
- Install mysql-community-server
```sudo yum install mysql-community-server```
- Start mysql server
```sudo systemctl start mysqld```
- Find default root password
```sudo grep 'temporary password' /var/log/mysqld.log```
- MySQL DB secure setting: Update root password, remove anonymous user,...
```mysql_secure_installation```
- Access to mysql sql console and create 2 databases and grant permission for them: drupal and civicrm
```
> create database drupal;
> create database civicrm;
> grant all privileges on drupal.* to ‘drupal’@’127.0.0.1’ identified by ‘drupal’;
> grant all privileges on civicrm.* to ‘civicrm’@’127.0.0.1’ identified by civicrm;
> grant SELECT on civicrm.* to ‘drupal’@’127.0.0.1’ identified by ‘drupal’;
```

## Setup, configure webserver and Install Drupal

- Download Drupal to your local and extract to /usr/share/drupal
```wget http://ftp.drupal.org/files/projects/drupal-7.32.tar.gz```

- Create document root:
```sudo mkdir -p /usr/share/drupal```

- Create temporary folder for let's encrypt
```sudo mkdir -p /usr/share/drupal/.well-known/acme-challenge```
- Set permission of document root folder:
```sudo chmod 750 /usr/share/drupal```

```sudo chown -R nginx:nginx /usr/share/drupal```

**Configure Nginx:**
- Create a nginx configure for drupal
```sudo touch /etc/nginx/conf.d/drupal.conf```

Paste the content below to drupal.conf

```
server {
    server_name tasks.rivieu.com;
    listen 80;
 
    root /usr/share/drupal;
    access_log /var/log/nginx/tasks-access.log;
    error_log /var/log/nginx/tasks-error.log;
    index index.php index.html index.htm;

    location /.well-known/acme-challenge {
        root /usr/share/letsencrypt;
    }
 
    location / {
        try_files  $uri $uri/ /index.php?$args;
    }
 
    error_page 404 /404.html;
        location = /40x.html {
    }
 
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
 
    location ~* \.php$ {
        try_files $uri = 404;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  }
 
}
``` 

**Restart Nginx Server:** 
```sudo systemctl restart nginx```

**Configure PHP-FPM**
- Open PHP-FPM configuration:
```sudo vi /etc/php-fpm.d/www.conf```
Find and replace these lines:
```
user = apache to user = nginx
group = apache to group = nginx
listen.owner = nobody to listen.owner = nginx
listen.group = nobody to listen.group = nginx
listen.mode = 0660
```
**Restart PHP-FPM and enable it:**
```
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```
**Open browse and access:**
```http://your_domain``` and follow the guide steps to finish install Drupal

# Enable HTTPS for Drupal site

We will use free SSL/TLS certificates from Let’s Encrypt (https://letsencrypt.org/)

- Download Let’s Encrypt Client (I use git to clone letsencrypt)
```sudo git clone https://github.com/certbot/certbot /opt/letsencrypt```
- Create temporary folder for the let’s encrypt temporary file
```sudo mkdir -p /usr/share/letsencrypt/```
- Create a file that use as a pattern for Cert /etc/letsencrypt/configs/tinhuynh.test.co

With the content of tinhuynh.test.co:
```
domains = tasks.rivieu.com
rsa-key-size = 2048 # Or 4096
server = https://acme-v01.api.letsencrypt.org/directory
email = vantintttp@gmail.com
text = True
authenticator = webroot
webroot-path = /usr/share/letsencrypt/
```
- Request the Certificate
- Change to /opt/letsencrypt folder
- Run cert request:
```./certbot-auto --config /etc/letsencrypt/configs/tinhuynh.test.co certonly```
**Enable HTTPS**
- Create a nginx configure for https
```sudo touch /etc/nginx/conf.d/drupal_https.conf```
- Paste the contents below to drupal_https.conf
```
server {
    server_name tasks.rivieu.com;
    listen 443 ssl default_server;
 
    ssl_certificate /etc/letsencrypt/live/tasks.rivieu.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tasks.rivieu.com/privkey.pem;
 
    root /usr/share/drupal;
    access_log /var/log/nginx/tasks-access.log;
    error_log /var/log/nginx/tasks-error.log;
    index index.php;
 
    location /.well-known/acme-challenge {
        root /usr/share/letsencrypt;
    }
    location / {
        try_files  $uri $uri/ /index.php?$args;
    }
    error_page 404 /404.html;
        location = /40x.html {
    }
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
    location ~* \.php$ {
        try_files $uri = 404;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  }
}
```
**Restart nginx:**
```sudo systemctl restart nginx```

**Open browse and access:**
```https://your_domain```

# Install CiViCRM

- Download civicrm modules and extract to folder /usr/share/drupal/sites/all/modules/civicrm
**Install by Run the Installer**
- Login to Drupal site with Administrator 
- Point your web browse to `http://your_domain/sites/all/modules/civicrm/install/index.php`
**Follow the guide steps to finish Install CiViCRM (Note: Make sure you enter different the database information of Drupal and CiViCRM that you create above).**

# Backup
- Create a script for backup `sudo vi /usr/share/scripts/script.sh`
```
#!/bin/bash
getDate=$(date +%Y-%m-%d)
tar -czvf /backup/drupal-”$getDate”.tar.gz /usr/share/drupal
```
- Set crontab for backup 
***add below lines to end of crontab file `sudo vi /etc/crontab`***
```
#Daily backup
00 00 * * * /usr/share/scripts/script.sh
#Weekly backup (Sunday)
00 01 * * 0 /usr/share/scripts/script.sh
#Monthly backup (on 29)
00 02 29 * * /usr/share/scripts/script.sh
```
# Auto Renew let’s encrypt Certificate
- Create a script and set execute permission for it: ```sudo vi /usr/share/scripts/certRenew.sh```
**add the contents below to certRenew.sh file**
```
#!/bin/sh
cd /opt/letsencrypt/
./certbot-auto --config /etc/letsencrypt/configs/tinhuynh.test.co renew
if [ $? -ne 0 ]
 then
        ERRORLOG=`tail /var/log/letsencrypt/letsencrypt.log`
        echo -e "The Let's Encrypt cert has not been renewed! n n" $ERRORLOG
 else
        nginx -s reload
fi
exit 0
```
- Set crontab for auto renew: 
***add below lines to end of crontab file `sudo vi /etc/crontab`***
```
# Auto Renew every 2 month
0 0 1 JAN,MAR,MAY,JUL,SEP,NOV * /usr/share/scripts/certRenew.sh
```
