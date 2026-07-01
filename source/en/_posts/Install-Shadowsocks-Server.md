---
title: Installing the Shadowsocks Server
date: 2016-11-08 10:23:20
lang: en
tags:
  - Server
  - Shadowsocks
categories:
  - Server
description: How to install and configure Shadowsocks on a server.
---
## Introduction

As the domestic "wall" grows ever "taller", mastering a convenient, fast, and affordable way to "leap over" it has become a necessity. Below we introduce [Shadowsocks](https://github.com/shadowsocks/shadowsocks "Shadowsocks"), a lightweight, cross-platform, open-source tool that is very easy to install and configure.

## Installation

### Prerequisites
* A VPS
    You can use the SFO region of [digitalocean](https://www.digitalocean.com)
* A Python environment

### Server Installation
 1. #### Install via pip
     * Install pip
       ```bash
       apt-get install python-pip
       ```
     * Install shadowsocks
          ```bash
          pip install shadowsocks
          ```

### Configuration
  ```bash
  vim /etc/shadowsocks.json
  ```
  Add the following content
  ```bash
  {
      "server": "my_server_ip",               # server IP
      "server_port": 8000,                    # listening port
      "local_address": "x.x.x.x",             # server local address
      "local_port": 1080,                     # server local port
      "password": "mypassword",               # connection password
      "timeout": 300,                         # connection timeout
      "method": "rc4-md5"
      "fast_open": true                       # whether to enable TCP_FASTOPEN (requires kernel support)
      "workers": 5                            # number of worker processes
  }
  ```

  ### System Optimization
  Confirm the kernel version is 3.7.1 or above
   ```bash
   uname -r

   # 4.4.0-45-generic
   ```
1. #### Maximum number of file descriptors
   * Before each Shadowsocks launch
     ```bash
     ulimit -SHn 51200
     ```
   * Take effect at system boot
     ```bash
     vim /etc/security/limits.conf
     # Add
     * soft nofile 51200
     * hard nofile 51200
     # First column: user or group
     # Second column: hard = hard limit, soft = soft limit. Generally soft is smaller than hard; exceeding soft triggers a warning, while hard is the ceiling
     # Third column: nofile = number of open files
     ```
     ```bash
     vim  /etc/pam.d/common-session
     # Add the line
     session required pam_limits.so
     ```
     ```bash
     vim /etc/profile
     # Append at the end of the file
     ulimit -SHn 51200
     ```
2. ####  Tune kernel parameters
     ```bash
     vim /etc/sysctl.conf

     # Add the configuration

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


      # Apply the configuration
      sysctl -p
     ```
   * TCP_FASTOPEN
     The Linux kernel version on both the server and client sides must be newer than 3.7.1
     ```bash
     # Check whether it is in effect
     sysctl net.ipv4.tcp_fastopen

     # net.ipv4.tcp_fastopen = 3
     ```
   * TCP congestion control algorithms
     Linux ships with several TCP congestion control algorithms.
     1. reno is the most basic congestion control algorithm and the experimental prototype of the TCP protocol.
     2. bic suits links with high RTT but extremely rare packet loss, such as the route between North America and Europe; it was the default algorithm for Linux kernels from 2.6.8 to 2.6.18.
     3. cubic is a modified version of bic and covers a broader range of scenarios than bic; it is the default algorithm for Linux kernels after 2.6.19.
     4. hybla suits networks with high latency and high packet loss rates, such as satellite links — and equally the route between China and the United States.

     ```bash
     # List algorithms supported by the system
     sysctl net.ipv4.tcp_available_congestion_control

     # net.ipv4.tcp_available_congestion_control = hybla cubic reno
     ```
### Launch
 * Launch directly
 ```bash
 ssserver -p 8000 -k password -m rc4-md5 -d {start | stop}
 ```
 * Launch from a configuration file
 ```bash
 ssserver -c /etc/shadowsocks.json -d {start | stop}
 ```
 ## Summary

With the digitalocean SFO2 region at 300+ ms latency, you can smoothly stream 1080P video on YouTube (Hunan Telecom).
