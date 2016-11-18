---
title: Ubuntu 下增加 Swap 
date: 2016-10-21 10:29:11
tags:
  - Ubuntu
  - Swap
  - Server
categories:
  - Server
description: 现在很多 Ubuntu 系统的 VPS 默认是不开启 Swap ，或者 Swap 大小不足。本文介绍如何在 Ubuntu 系统下增加 Swap。
---
## 前言

### 什么是Swap
   Linux 系统中的 Swap ，又名交换空间。是类似于 Windows 系统的虚拟内存 (pagefile.sys)，当内存不够用的时候会把一部分硬盘空间虚拟成内存，用来存储内存中暂时不使用的数据。
   
## 安装
1. #### 查看
   ```bash
   free -m
   ```
   ```bash
                total       used       free     shared    buffers     cached
   Mem:          3952       2035       1916          9        217       1392
   -/+ buffers/cache:        425       3526
   Swap:            0          0          0
   ```
   可以看到 Swap 没有开启，下面我们来将其增加到内存大小 (4G)。
   
2. #### 创建 Swap 文件
    ```bash
    mkdir swap
    
    cd swap
    
    sudo dd if=/dev/zero of=swapfile bs=1024 count=4M   # bs 表示块大小  count 表示块数目  1024 * 4M  = 4G
    # 4194304+0 records in
    # 4194304+0 records out
    # 4294967296 bytes (4.3 GB) copied, 88.3999 s, 48.6 MB/s
    ```
    把文件转换成 Swap 文件
    ```bash
    sudo mkswap -f  swapfile 
    # Setting up swapspace version 1, size = 4194300 KiB
    # no label, UUID=bebbcbad-dda2-49f9-9aab-4b24b1d62d87
    ```
3. #### 激活 Swap 
    ```bash
    sudo swapon swapfile
    ```
     查看激活情况
    ```bash
    free -m  
    ```
    ```bash
                 total       used       free     shared    buffers     cached
    Mem:          3952       3842        109          9          1       3369
    -/+ buffers/cache:        470       3481
    Swap:         4095          0       4095
    ```
4. #### 配置
    * 修改swappiness
      swappiness 为 0 - 100的数值，数值越大表示越积极的去使用 Swap。
      
      - 暂时修改
        ```bash
        sudo sysctl vm.swappiness=40
        ```
      - 永久修改
         ```bash
         sudo vim /etc/sysctl.conf
         # 增加一行
         vm.swappiness = 40
         ```
    * 修改权限
         设置成只有 root 用户具有读写权限
         ```bash
         sudo chown root:root /swap/swapfile
         sudo chmod 0600 /swap/swapfile
         ```
5. #### 开机启动
    ```bash
    sudo vim /etc/fstab
    ```
    在文件最后面添加
    ```bash
    /swap/swapfile       none    swap    sw      0       0
    ```