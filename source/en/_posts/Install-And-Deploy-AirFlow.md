---
title: Installing and Deploying AirFlow
date: 2017-01-19 16:44:02
tags:
- AirFlow
- Server
- Supervisord
categories: Server
description: A brief introduction to installing and deploying AirFlow.
lang: en
---

## Introduction

AirFlow is an open-source workflow scheduler written in Python that ships with a rich UI.

## Installation

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

## Configuration

### AirFlow

#### Initialization

```shell
airflow initdb
```

#### Adding User Login

Install the corresponding module.

```
pip install "airflow[password]"
```

Add the configuration.

```
vim airflow.cfg
## Under [webserver], add
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth
```

Switch into the airflow directory.

```
cd ~/airflow
python
```

Run the Python commands.

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

Add startup management for the webserver and scheduler.

```
vim /etc/supervisor/conf.d/airflow.conf 

## Add
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

## Issues

```
ImportError: No module named pidlockfile

## Solution

aptitude remove python-lockfile
pip install lockfile
ImportError: cannot import name MySqlOperator

## Solution

pip install airflow[celery]
```
