---
title: PHP-FPM 进程管理模式简介
date: 2016-10-10 16:02:10
tags:
  - Server
  - PHP
  - Process Manager
categories:
  - Server
description: 
  - 在服务器运行时，会出现服务器出现PHP-FPM内存占用过大、PHP-FPM进程数目太多等相关问题导致网站或者服务器访问无法问题，那PHP-FPM的进程管理模式到底是什么样子的，下面我们来大概了解下。
---
## 前言
在服务器运行时，会出现服务器出现PHP-FPM内存占用过大、PHP-FPM进程数目太多等相关问题导致网站或者服务器访问无法问题，那PHP-FPM的进程管理模式到底是什么样子的，下面我们来大概了解下。
## 简介
通过查看一下配置文件，我们可以发现PHP-FPM具有三种管理模式`{ static | dynamic | ondemand}`
```bash
vim /etc/php/7.0/fpm/pool.d/www.conf 
```
```bash
; Choose how the process manager will control the number of child processes.
; Possible Values:
;   static  - a fixed number (pm.max_children) of child processes;
;   dynamic - the number of child processes are set dynamically based on the
;             following directives. With this process management, there will be
;             always at least 1 children.
;             pm.max_children      - the maximum number of children that can
;                                    be alive at the same time.
;             pm.start_servers     - the number of children created on startup.
;             pm.min_spare_servers - the minimum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is less than this
;                                    number then some children will be created.
;             pm.max_spare_servers - the maximum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is greater than this
;                                    number then some children will be killed.
;  ondemand - no children are created at startup. Children will be forked when
;             new requests will connect. The following parameter are used:
;             pm.max_children           - the maximum number of children that
;                                         can be alive at the same time.
;             pm.process_idle_timeout   - The number of seconds after which
;                                         an idle process will be killed.
; Note: This value is mandatory.
```
## 详解
### Static
`pm = static`
启动时创建并始终保持一个固定数量的子进程。
```bash
pm.max_children        # 限定php-fpm的最大进程数
```
### Dynamic
`pm = dynamic  (默认)` 
启动时会创建固定数目`start_servers`的子进程，使用过程中会新建子进程但数目不会超过最大子进程数`max_children`，子进程数目会在`start_servers`和`max_children`之间波动。而闲置的子进程数由`min_spare_servers` 和 `max_spare_servers` 进行控制。
```bash
pm.max_children        # 限定php-fpm的最大进程数
pm.start_servers       # 起始php-fpm进程数量。
pm.min_spare_servers   # 空闲状态下的最小php-fpm进程数量。
pm.max_spare_servers   # 空闲状态下的最大php-fpm进程数量。（max_spare_servers <= max_children）

#官方建议值
start_servers = min_spare_servers + (max_spare_servers - min_spare_servers) / 2
```
### Ondemand
`pm = ondemand ` 
启动时不会创建子进程，当有请求时创建进程，数目不超过`max_children`。当进程空闲时间超过`process_idle_timeout`将被释放
```bash
pm.max_children           # 限定php-fpm的最大进程数
pm.process_idle_timeout   # 进程闲置多少秒后被释放
```

## 建议
 - 配置过程中我们可以采用 `内存/30M (单个进程最大内存数)`的数目来设置`max_children`
 - `static` 适用于大内存服务器，因为子进程在创建和释放时都会消耗服务器资源 (时间、内存、CPU...)
 - `dynamic` 适用于小内存服务器，方式灵活，节省内存。
 - `ondemand` 不建议使用在大访问量应用，大量创建和释放进程开销很大
 - 生成环境建议采用 `static` 模式


