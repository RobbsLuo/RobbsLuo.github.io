---
title: 安装和部署AirFlow
date: 2017-01-19 16:44:02
tags:
- AirFlow
- Server
- Supervisord
categories: Server
description: 简单介绍如何安装和部署AirFlow
---

## 简介

AirFlow 是采用 Python 编写的一个开源工作流调度器，它有一个丰富的UI。

## 安装

### Python

```Shell
aptitude install python
aptitude install python-dev
aptirude install python-pip
aptitude install libmysqlclient-dev
```

### AirFlow

```shell
pip install airflow
```

### Supervisor

```shell
aptitude install supervisor
```

## 配置

### AirFlow

#### 初始化

```shell
airflow initdb
```

#### 添加用户登录

安装相应模块

```
pip install "airflow[password]"
```

添加配置

```
vim airflow.cfg
## 在 [webserver]下 加入
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth
```

进入airflow目录下

```
cd ~/airflow
python
```

运行Python命令

```
import airflow
from airflow import models, settings
from airflow.contrib.auth.backends.password_auth import PasswordUser
user = PasswordUser(models.User())
user.username = 'user_name'
user.email = 'email@example.com'
user.password = 'password'
session = settings.Session()
session.add(user)
session.commit()
session.close()
exit()
```

#### Supervisord

加入 webserver 和 scheduler 启动管理

```
vim /etc/supervisor/conf.d/airflow.conf 

## 加入
[program:airflow_webserver]
command=airflow webserver
user=ubuntu
stderr_logfile=/var/log/airflow/webserver.err.log
stdout_logfile=/var/log/airflow/webserver.out.log
[program:airflow_scheduler]
command=airflow scheduler
user=ubuntu
stderr_logfile=/var/log/airflow/scheduler.err.log
stdout_logfile=/var/log/airflow/scheduler.out.log
```

## 问题

```
ImportError: No module named pidlockfile

## 解决方案

aptitude remove python-lockfile
pip install lockfile
ImportError: cannot import name MySqlOperator

## 解决方案

pip install airflow[celery]
```