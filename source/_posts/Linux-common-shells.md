---
title: Linux 常用命令
date: 2016-09-23 19:14:35
tags:
  - Linux
  - Shell
categories:
  - Linux
description:
  - 整理了一些自己比较常用的命令，后期将持续更新。
---
## 前言
整理了一些自己比较常用的命令，后期将持续更新。
## 详解
### 文件命令
``` bash
$ ls                 ## 查看目录下的子节点（文件夹和文件）信息  -a:显示隐藏文件  -l:列表显示
$ pwd                ## 查看当前所在的工作目录的全路径
$ cd dir             ## 打开目录 - 前一个工作目录 ~ 当前用户目录
$ rm                 ## 删除文件  -r  删除目录 -f 强制删除文件
$ cp file1 file2     ## 复制  -r  赋值目录，没有就创建
$ ln -s file link    ## 创建file的符号连接link
$ touch file         ## 创建文件
$ mkdir dir          ## 创建目录
$ more file          ## 查看file内容
$ tail -f file       ## 从后10行开始查看file
$ chmod octal file   ## 更改file权限  777 755
$ chown              ## 更改文件所有者
$ Ctrl + W           ## 删除当前行中的字
$ Ctrl + U           ## 删除整行
```
### 进程管理
```bash
$ ps            ## 现实当前的活动进程
$ top           ## 显示所有正在运行的进程
$ kill pid      ## 杀掉进程 pid
$ killall proc  ## 杀掉所有名位为 proc 的进程
$ Ctrl + C      ## 停止当前命令
$ Ctrl + \      ## 停止当前命令，当 Ctrl + C 不好用的时候
$ Ctrl + Z      ## 停止当前命令，并使用 fg 恢复
$ Ctrl + D      ## 注销当前会话，与 exit 相似
$ !!            ## 重复上次的命令
$ fg            ## 将进程转到前台
$ bg pid        ## 将进程转到前台
```
### SSH
```bash
$ ssh user@host         ## 以user去登录host   -p 端口 -i key文件 -vvv Debug模式mkdir
$ ssh-copy-id user@host ## 将密钥添加到host实现无密码登录
```
### 系统信息
```bash
$ date                   ## 显示当前日期和时间
$ ntpdate ntp.ubuntu.com ## 与Ubuntu NTP server同步时间
$ cal                    ## 显示当月日历
$ uptime                 ## 显示系统从开机到现在所运行的时间
$ getent passwd          ## 所有用户列表
$ w                      ## 显示登录的用户
$ whoami                 ## 查看当前用户名
$ finger user            ## 显示user相关信息
$ uname -a               ## 显示内核信息
$ cat /proc/cpuinfo      ## 显示cpu信息
$ cat /proc/meminfo      ## 显示内存信息
$ man command            ## 显示command 的说明手册
$ df                     ## 显示磁盘占用
$ du                     ## 显示目录空间占用
$ free                   ## 显示内存及交换区占用情况
```
### 网络
```bash
$ ping host                      ## ping host 并输出结果
$ nmap host                      ## 扫描网络寻找开放的端口
$ whois domain                   ## 获取 domain 的 whois 信息
$ dig domain                     ## 获取 domain 的 DNS 信息
$ dig -x domain                  ## 逆向查询 host
$ wget file                      ## 下载file
$ wget -c file                   ## 断点续传
$ curl -I http://www.q.com       ## 获取HTTP头信息
```
### 安装
```bash
$ ./configure
$ make
$ make install
```
### 搜索
```bash
$ grep
$ find
```
