---
title:   采用Let's Encrypt 证书在Nginx上实现 HTTPS
date: 2016-10-11 11:09:50
tags:
  - Ubuntu
  - HTTPS
  - Nginx
  - Server
categories:
  - Server
description:  简单介绍怎么采用Let's Encrypt 推荐的工具Certbot在Nginx上实现Https
---
## 前言

### 什么是HTTPS
  HTTPS（全称：Hyper Text Transfer Protocol over Secure Socket Layer），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。
  
### 为什么使用HTTPS
  我们大致也明白了HTTPS其实就是一个经过加密后的HTTP，具有更高的安全性。
  
### 证书选择
目前市面上比较成熟的HTTPS 证书提供商：
- 做证书起家的[VeriSign](http://www.verisign.com/ "VeriSign")
- 老牌的 [StartSSL](https://www.startssl.com/ "StartSSL")
- 新星 [Let’s Encrypt](https://letsencrypt.org/ "Let’s Encrypt")
- 国内的 [沃通](https://www.wosign.com/ "沃通")
- ......
 
本文将使用Let's Encrypt证书，因为它免费、快速、可自动化续签......

## 实践
### 需要工具
   - Ubuntu16.04的服务器 
   - 域名
 
### 安装
  根据Let’s Encrypt上官方推荐我们将采用自动化工具[Certbot](https://certbot.eff.org/ 'Certbot')，并安装推荐教程直接操作。

1.  安装Certbot   
```bash
sudo apt-get install letsencrypt
```
2. 申请证书
```bash
letsencrypt certonly --agree-tos --email xx@xx.com --webroot -w /var/www/example -d example.com -d www.example.com -w /var/www/thing -d thing.is -d m.thing.is
# --email 后面替换成自己的Email用来接受通知
# -d 填写需要Https访问的域名，可多个
```
  命令运行完以后会在`/etc/letsencrypt/live/example.com/`目录下生成一下文件
  - cert.pem  服务器证书文件
  - chain.pem
  - fullchain.pem =  cert.pem + chain.pem  `Nginx配置文件中的 ssl_certificate`
  - privkey.pem `Nginx 配置文件中的 ssl_certificate_key`
3. 配置Nginx
  ```bash
  vim /etc/nginx/sites-available/default
  ```
  ```bash
server {
    listen 80;
    # listen [::]:80;
    server_name example.com;

    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    return 301 https://$host$request_uri;
}

server {
    # SSL configuration
    #
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name example.com;

    location / {
        root   /var/www/html;
        index  index.html index.htm;
    }
    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    # ssl_dhparam /etc/ssl/certs/dhparam.pem;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    # ssl_trusted_certificate /etc/letsencrypt/live/walfud.com/root_ca_cert_plus_intermediates;

    # resolver 8.8.8.8 8.8.4.4 valid=300s;
    # resolver_timeout 5s;
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        # With php7.0-cgi alone:  
        fastcgi_pass 127.0.0.1:9000;
        # With php7.0-fpm:
        # fastcgi_pass unix:/var/run/php7.0-fpm.sock;
    }
}
```
4. 重启Nginx
  ```bash
  service nginx reload
  ```
5. 自动续签
    Let’s Encrypt 证书的有效期是 90 天，为了保证证书的有效性，我们需要及时的去续签(renew) 证书。这里我们采用`cron`来进行定期的续签。`letsencrypt renew`只会在证书过期的时候才会刷新证书，如果证书没有过期将不会执行任何命令。
  ```bash
  crontab -e 
  ```
  ```bash
  # 每天夜里凌晨 0 点续签:
  * 0 * * * letsencrypt renew

  # 重启 nginx 以使证书生效
  * 1 * * * service nginx reload
  ```


