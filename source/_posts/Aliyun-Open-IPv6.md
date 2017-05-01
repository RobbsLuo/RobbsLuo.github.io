---
title: 阿里云开启 IPv6 支持
date: 2017-05-01 23:36:43
tags:
 - Aliyun
 - IPv6
categories:
 - Server
description: 开启阿里云的 IPv6 支持
---

## 开启限制

```shell
vim /etc/sysctl.conf

# 增加配置
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0

```

## 查看是否生效

```shell
ifconfig

# 信息
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 114.55.127.xxx  netmask 255.255.252.0  broadcast 114.55.119.255
        inet6 fe80::219:3eff:fexx:32ec  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:01:49:37  txqueuelen 1000  (Ethernet)
        RX packets 68348  bytes 41664492 (39.7 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 18859  bytes 3632548 (3.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

`fe80::219:3eff:fexx:32ec` 是IPv6地址。