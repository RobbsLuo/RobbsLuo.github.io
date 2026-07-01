---
title: Implementing HTTPS on Nginx with a Let's Encrypt Certificate
date: 2016-10-11 11:09:50
lang: en
tags:
  - Ubuntu
  - HTTPS
  - Nginx
  - Server
categories:
  - Server
description: A brief guide to implementing HTTPS on Nginx with Certbot, the tool recommended by Let's Encrypt.
---
## Preface

### What is HTTPS
  HTTPS (Hyper Text Transfer Protocol over Secure Socket Layer) is an HTTP channel aimed at security. Simply put, it is the secure version of HTTP. It adds an SSL layer on top of HTTP, and since the security foundation of HTTPS is SSL, the encryption details rely on SSL.

### Why use HTTPS
  We can roughly understand that HTTPS is essentially an encrypted form of HTTP, which provides higher security.

### Choosing a certificate
The more mature HTTPS certificate providers currently on the market include:
- [VeriSign](http://www.verisign.com/ "VeriSign"), which built its business on certificates
- The well-established [StartSSL](https://www.startssl.com/ "StartSSL")
- The rising star [Let's Encrypt](https://letsencrypt.org/ "Let's Encrypt")
- China-based [WoSign](https://www.wosign.com/ "WoSign")
- ......

This article will use a Let's Encrypt certificate because it is free, fast, and supports automated renewal...

## Practice
### Required tools
   - An Ubuntu 16.04 server
   - A domain name

### Installation
  Following the official recommendation from Let's Encrypt, we will use the automation tool [Certbot](https://certbot.eff.org/ 'Certbot') and follow the installation tutorial directly.

1.  Install Certbot
```bash
sudo apt-get install letsencrypt
```
2. Apply for a certificate
```bash
letsencrypt certonly --agree-tos --email xx@xx.com --webroot -w /var/www/example -d example.com -d www.example.com -w /var/www/thing -d thing.is -d m.thing.is
# Replace the value after --email with your own email address to receive notifications
# -d specifies the domains that require HTTPS access; multiple domains are supported
```
  After the command finishes, the following files will be generated in the `/etc/letsencrypt/live/example.com/` directory:
  - cert.pem  server certificate file
  - chain.pem
  - fullchain.pem = cert.pem + chain.pem  corresponds to `ssl_certificate` in the Nginx configuration
  - privkey.pem corresponds to `ssl_certificate_key` in the Nginx configuration
3. Configure Nginx
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
4. Reload Nginx
  ```bash
  service nginx reload
  ```
5. Automatic renewal
    A Let's Encrypt certificate is valid for 90 days. To keep the certificate valid, we need to renew it in a timely manner. Here we use `cron` to schedule periodic renewals. `letsencrypt renew` only refreshes the certificate when it is close to expiry; if the certificate has not expired, it does nothing.
  ```bash
  crontab -e
  ```
  ```bash
  # Renew every day at midnight:
  * 0 * * * letsencrypt renew

  # Reload nginx to make the new certificate take effect
  * 1 * * * service nginx reload
  ```
