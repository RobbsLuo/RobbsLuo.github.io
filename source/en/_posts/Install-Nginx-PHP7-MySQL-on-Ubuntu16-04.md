---
title: Installing Nginx, PHP7, and MySQL 5.7 on Ubuntu 16.04
date: 2016-09-30 12:04:51
tags:
  - Server
  - Ubuntu
  -  Nginx
categories:
  - Server
lang: en
description:  How to install Nginx, PHP7, and MySQL 5.7 on Ubuntu 16.04 via PPA.
---

## Prerequisites
Before installing, first add the corresponding [PPA](/2016-09-30/Software-PPA-Sources.html "Common Software PPA Sources") repositories.
## Installation
### Nginx
```bash
aptitude install nginx -y
```
### MySQL
```bash
aptitude install mysql-server-5.7 -y
```
### PHP7.0
```bash
aptitude install php7.0 php7.0-mysql php7.0-mcrypt php7.0-mbstring php7.0-fpm php7.0-cli php7.0-xml php7.0-zip -y
aptitude install composer
```

## Configuration
### PHP
```bash
### vim /etc/php/7.0/fpm/pool.d/www.conf
### Change the Unix socket to a TCP socket
listen = 127.0.0.1:9000
```
### Nginx
```bash
### vim /etc/nginx/sites-available/default

root /var/www/html;
# Add index.php to the list if you are using PHP
index index.php index.html index.htm;
server_name _;
location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        # With php7.0-cgi alone:
        fastcgi_pass 127.0.0.1:9000;
        # With php7.0-fpm:
        # fastcgi_pass unix:/var/run/php7.0-fpm.sock;
}
```

## Misc
### Related Commands
```bash
service php7.0-fpm {start|stop|status|restart|reload|force-reload}
service mysql {start|stop|restart|reload|force-reload|status}
service nginx {start|stop|restart|reload|force-reload|status|configtest}
```
