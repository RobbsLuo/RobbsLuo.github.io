---
title: Ubuntu16.04 安装 MongoDB
date: 2017-04-25 15:03:36
tags:
 - MongoDB
 - Ubuntu
categories:
 - Server
description: 介绍在 Ubuntu 16.04上安装 MongoDB, 并增加帐号密码密码访问
---

## 添加软件源

```shell
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6


echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-3.4.list
```

## 安装MongoDB

```Sh
apt-get update
apt-get install mongodb-org -y
```

## 启动

```shell
service mongod start
service mongod stop
```

## 创建帐号密码

```shell
# 添加配置文件
vim /ect/mongod.conf

security:  
authorization: "enabled"

# 重启服务
service mongod stop
service mongod start

# 进入数据库
mongo
> use admin
> db.createUser({user:"user_test",pwd:"pwd_test",roles:["root"]}) # 创建账号  
> db.auth("user_test","pwd_test") # 就可以进入了
```

