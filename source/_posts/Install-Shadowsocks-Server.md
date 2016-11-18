---
title: 安装 Shadowsocks 服务器端
date: 2016-11-08 10:23:20
tags:
  - Server
  - Shadowsocks
categories:
  - Server
description: 如何在服务器上安装并配置 Shadowsocks 。
---
## 前言
随着国内『墙』的越来越『高』，掌握一门方便、快捷、便宜的『飞檐走壁』技巧成为了刚需。下面我们介绍一款轻量、多平台、安装配置十分简单的开源软件 [Shadowsocks](https://github.com/shadowsocks/shadowsocks "Shadowsocks") 。

## 安装
### 准备工作
* VPS 一台
    可以使用 [digitalocean](https://www.digitalocean.com) 的 SFO 机房
* Python 环境

### 服务器安装
 1. #### pip 安装法
     * 安装 pip
       ```bash
       apt-get install python-pip
       ```
     * 安装 shadowsocks
          ```bash
          pip install shadowsocks
          ```
          
### 配置
  ```bash
  vim /etc/shadowsocks.json
  ```
  加入以下内容
  ```bash
  {
      "server": "my_server_ip",               # 服务器IP
      "server_port": 8000,                    # 开启端口
      "local_address": "x.x.x.x",             # 服务器本地地址
      "local_port": 1080,                     # 服务器本地端口
      "password": "mypassword",               # 连接密码
      "timeout": 300,                         # 连接超时时间
      "method": "rc4-md5"
      "fast_open": true                       # 是否开启TCP_FASTOPEN，需要系统开启支持
      "workers": 5                            # 进程数
  }
  ```
  
  ### 系统优化
  确认系统内核是 3.7.1 以上
   ```bash
   uname -r
   
   # 4.4.0-45-generic
   ```
1. #### 系统文件描述符的最大限数
   * Shadowsocks 每次启动前
     ```bash
     ulimit -SHn 51200
     ```
   * 系统启动时生效
     ```bash
     vim /etc/security/limits.conf
     # 添加
     * soft nofile 51200
     * hard nofile 51200
     # 第一列 是用户或者用户组
     # 第二列 hard: 硬限制，soft: 软件限制，一般来说 soft 要比 hard 小，超过 soft 报警，hard 是底线
     # 第三列 nofile 打开文件数
     ```
     ```bash
     vim  /etc/pam.d/common-session
     #添加一行
     session required pam_limits.so
     ```
     ```bash
     vim /etc/profile
     # 在文件末尾加入
     ulimit -SHn 51200
     ```
2. ####  调整内核参数
     ```bash
     vim /etc/sysctl.conf

     # 添加配置

     fs.file-max = 51200                            # max open files

      net.core.rmem_max = 67108864                  # max read buffer
      net.core.wmem_max = 67108864                  # max write buffer
      net.core.netdev_max_backlog = 250000          # max processor input queue
      net.core.somaxconn = 4096                     # max backlog

      net.ipv4.tcp_syncookies = 1                   # resist SYN flood attacks
      net.ipv4.tcp_tw_reuse = 1                     # reuse timewait sockets when safe
      net.ipv4.tcp_tw_recycle = 0                   # turn off fast timewait sockets recycling
      net.ipv4.tcp_fin_timeout = 30                 # short FIN timeout
      net.ipv4.tcp_keepalive_time = 1200            # short keepalive time
      net.ipv4.ip_local_port_range = 10000 65000    # outbound port range
      net.ipv4.tcp_max_syn_backlog = 8192           # max SYN backlog
      net.ipv4.tcp_max_tw_buckets = 5000            # max timewait sockets held by system simultaneously
      net.ipv4.tcp_rmem = 4096 87380 67108864       # TCP receive buffer
      net.ipv4.tcp_wmem = 4096 65536 67108864       # TCP write buffer
      net.ipv4.tcp_mtu_probing = 1                  # turn on path MTU discovery

      net.ipv4.tcp_fastopen = 3                     # enable TCP_FASTOPEN 

      net.ipv4.tcp_congestion_control = hybla


      #使配置生效
      sysctl -p
     ```
   * TCP_FASTOPEN
     服务端和客户端Linux内核版本必须新于 3.7.1
     ```bash
     # 查看是否生效
     sysctl net.ipv4.tcp_fastopen
     
     # net.ipv4.tcp_fastopen = 3
     ```
   * TCP拥塞控制算法
     Linux 中提供了多套TCP拥塞控制算法。
     1. reno是最基本的拥塞控制算法，也是TCP协议的实验原型。
     2. bic适用于rtt较高但丢包极为罕见的情况，比如北美和欧洲之间的线路，这是2.6.8到2.6.18之间的Linux内核的默认算法。
     3. cubic是修改版的bic，适用环境比bic广泛一点，它是2.6.19之后的linux内核的默认算法
     4. hybla适用于高延时、高丢包率的网络，比如卫星链路——同样适用于中美之间的链路。
     
     ```bash
     # 查看系统支持的算法
     sysctl net.ipv4.tcp_available_congestion_control
     
     # net.ipv4.tcp_available_congestion_control = hybla cubic reno
     ```
### 启动
 * 直接启动
 ```bash
 ssserver -p 8000 -k password -m rc4-md5 -d {start | stop}
 ```
 * 配置文件启动
 ```bash
 ssserver -c /etc/shadowsocks.json -d {start | stop}
 ```
 ## 小结
 采用 digitalocean 的 SFO2 机房 300+ms 延迟的情况下，能够很流畅的查看Youtube的 1080P （湖南电信）