---
title: Adding Swap on Ubuntu
date: 2016-10-21 10:29:11
tags:
  - Ubuntu
  - Swap
  - Server
categories:
  - Server
description: Many VPS instances running Ubuntu do not enable Swap by default, or their Swap size is insufficient. This article describes how to add swap space on Ubuntu.
lang: en
---
## Introduction

### What Is Swap
   Swap in a Linux system, also known as swap space, is similar to the virtual memory (pagefile.sys) of Windows. When memory runs low, a portion of the disk space is turned into virtual memory to store data that is not being used at the moment.
   
## Setup
1. #### Check the current state
   ```bash
   free -m
   ```
   ```bash
                total       used       free     shared    buffers     cached
   Mem:          3952       2035       1916          9        217       1392
   -/+ buffers/cache:        425       3526
   Swap:            0          0          0
   ```
   As you can see, Swap is not enabled. Below we will increase it to match the memory size (4G).
   
2. #### Create the Swap file
    ```bash
    mkdir swap
    
    cd swap
    
    sudo dd if=/dev/zero of=swapfile bs=1024 count=4M   # bs is the block size, count is the number of blocks; 1024 * 4M = 4G
    # 4194304+0 records in
    # 4194304+0 records out
    # 4294967296 bytes (4.3 GB) copied, 88.3999 s, 48.6 MB/s
    ```
    Convert the file into a swap file.
    ```bash
    sudo mkswap -f  swapfile 
    # Setting up swapspace version 1, size = 4194300 KiB
    # no label, UUID=bebbcbad-dda2-49f9-9aab-4b24b1d62d87
    ```
3. #### Activate Swap 
    ```bash
    sudo swapon swapfile
    ```
    Verify the activation.
    ```bash
    free -m  
    ```
    ```bash
                 total       used       free     shared    buffers     cached
    Mem:          3952       3842        109          9          1       3369
    -/+ buffers/cache:        470       3481
    Swap:         4095          0       4095
    ```
4. #### Configuration
    * Adjust swappiness
      swappiness is a value from 0 to 100. A higher value means the system will more actively use Swap.
      
      - Temporary change
        ```bash
        sudo sysctl vm.swappiness=40
        ```
      - Permanent change
         ```bash
         sudo vim /etc/sysctl.conf
         # Add a line
         vm.swappiness = 40
         ```
    * Change permissions
         Set the file so that only the root user has read and write permissions.
         ```bash
         sudo chown root:root /swap/swapfile
         sudo chmod 0600 /swap/swapfile
         ```
5. #### Enable on boot
    ```bash
    sudo vim /etc/fstab
    ```
    Add the following line at the end of the file.
    ```bash
    /swap/swapfile       none    swap    sw      0       0
    ```
