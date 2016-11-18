---
title: Ubuntu 上安装使用 DynamoDB
date: 2016-11-16 10:28:46
tags:
  - DynamoDB
  - Software
categories:
  -  Software
description: 记录在Ubuntu 上安装 DynamoDB
---

## 安装
1. #### Java SDK
   ```bash
    aptitude install openjdk-8-jdk
   ```

2. ####   安装解压软件
   ```bash    
    # 安装解压软件
    aptitude install unzip
   ```

3. #### 下载安装 DynamoDB
   ```bash
    # 下载压缩包
    wget http://dynamodb-local.s3-website-us-west-2.amazonaws.com/dynamodb_local_latest.zip 
    
    # 解压
    unzip ./dynamodb_local_latest.zip 
    
    # 启动
    java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb
    
    # 查看帮助
    java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -help
   ```

4. #### javascript shell
   [http://127.0.0.1:8000/shell/](http://127.0.0.1:8000/shell/)
