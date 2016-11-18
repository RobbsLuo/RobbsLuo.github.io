---
title: Siege 简介
date: 2016-10-24 14:46:29
tags:
  -  Server
  -  Siege
categories:
  -  Software
description: 简单的介绍下压力测试工具 Siege 的参数
---
## 前言
  [Siege](https://github.com/JoeDog/siege 'SIege') 是Linux 下一款压力测试和评测工具，设计用于 WEB 开发这评估应用在压力下的承受能力；可以配置针对一个 WEB 站点进行多用户的并发访问，记录每个用户所有请求过程的响应时间，并在一定数量的并发访问下重复进行。支持多链接，支持 GET 和 POST 请求。
  
## 安装
 * Mac OSX
   ```bash
    brew install siege
   ```
 * Ubuntu
   ```bash
    aptitude install siege
   ```

## 参数介绍 
```bash  
-c:  模拟有Ｎ个用户在并发访问 

-r:  重复测试运行Ｎ次 
-t:  持续测试时间   默认为分钟   5s(持续5秒)  5 (持续5分钟)
# -r和-t一般不同时使用

-f:  任务的URL列表  
-i:   随机访问-f指定的url.txt中的url列表项，以此模拟真实的访问情况(随机性)  

-b:  进行压力测试，请求无需等待   delay=0

-A:  指定访问的 User-Agent 
-H:  指定访问的 Header
-T:  指定访问的 Content-Type
```
 > ` siege -c 200 -r 100 http://www.google.com `
 > `siege  -c 200 -r 100 -f urls.txt`
 > `siege  -c 200 -r 100 -f urls.txt -i `
 > `siege -c 200 -r 100 -f urls.txt -i -b` delay=0，更准确的压力测试，而不是功能测试
 > `siege -H "Content-Type:application/json" -c 200 -r 100 -f urls.txt -i -b`

## 结果说明
 
  ```bash
  Transactions:		            20 hits            # 总共测试测试
  Availability:		            100.00 %           # 成功次数比
  Elapsed time:		            1.52 secs          # 总共耗时
  Data transferred:	            0.80 MB            # 总共数据传输
  Response time:		            0.14 secs          # 响应耗时
  Transaction rate:	            13.16 trans/sec    #  每秒处理请求数
  Throughput:		            0.52 MB/sec        # 吞吐率
  Concurrency:		            1.82               # 最高并发
  Successful transactions:            20                 # 成功请求数
  Failed transactions:                0                  # 失败请求数
  Longest transaction:                0.51               # 每次传输的最长时间
  Shortest transaction:               0.05               # 每次传输的最短时间
```

## Tips
  > 发送 POST 请求时，URL 格式为：`http://www.xxxx.com/ POST p1=v1&p2=v2`

  >  如果 URL 中含有空格和中文，要先进行编码

