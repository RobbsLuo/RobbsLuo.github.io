---
title: Install MongoDB on Ubuntu 16.04
date: 2017-04-25 15:03:36
tags:
 - MongoDB
 - Ubuntu
categories:
 - Server
description: How to install MongoDB on Ubuntu 16.04 and add username/password access control.
lang: en
---

## Add the Software Source

```shell
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6


echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-3.4.list
```

## Install MongoDB

```Sh
apt-get update
apt-get install mongodb-org -y
```

## Start the Service

```shell
service mongod start
service mongod stop
```

## Create a Username and Password

```shell
# Edit the configuration file
vim /ect/mongod.conf

security:  
authorization: "enabled"

# Restart the service
service mongod stop
service mongod start

# Enter the database
mongo
> use admin
> db.createUser({user:"user_test",pwd:"pwd_test",roles:["root"]}) # Create the account  
> db.auth("user_test","pwd_test") # You can now log in
```
