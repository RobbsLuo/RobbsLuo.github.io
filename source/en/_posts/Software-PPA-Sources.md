---
title: Common Software PPA Sources
date: 2016-09-30 11:40:37
tags:
  - Ubuntu
  - Software
  - PPA
  - Sources
categories:
  - Software
description: A backup of the PPA sources I use frequently; this post will be updated over time.
lang: en
---

## Introduction

A backup of the PPA sources I use frequently; this post will be updated over time.

## Details

### Nginx
```bash
add-apt-repository ppa:nginx/stable
## /etc/apt/sources.list
deb http://ppa.launchpad.net/nginx/stable/ubuntu xenial main 

apt-add-repository ppa:nginx/development
## /etc/apt/sources.list
deb http://ppa.launchpad.net/nginx/development/ubuntu xenial main 
```

### MySQL
```bash
add-apt-repository ppa:ondrej/mysql-5.7
## /etc/apt/sources.list
deb http://ppa.launchpad.net/ondrej/mysql-5.7/ubuntu xenial main 
```

### PHP70
```bash 
add-apt-repository ppa:ondrej/php
## /etc/apt/sources.list
deb http://ppa.launchpad.net/ondrej/php/ubuntu xenial main
```

### Git
```bash
add-apt-repository ppa:git-core/ppa
## /etc/apt/sources.list
deb http://ppa.launchpad.net/git-core/ppa/ubuntu xenial main 
deb-src http://ppa.launchpad.net/git-core/ppa/ubuntu xenial main 
```


## Tips
```bash
UnicodeDecodeError: 'ascii' codec can't decode byte 0xc5 in position 92: ordinal not in range(128)
```

Solution:
```bash
LC_ALL=C.UTF-8 add-apt-repository xxx
```

***
```bash
The program 'add-apt-repository' is currently not installed. You can install it by typing:
apt-get install software-properties-common
```

Solution:
```bash
apt-get install software-properties-common
```
