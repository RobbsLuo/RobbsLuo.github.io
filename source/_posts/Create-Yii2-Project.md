---
title: 创建 Yii2 项目
date: 2016-10-25 22:07:50
tags:
  - PHP
  - Yii2
categories:
  -  Develop
description: 简单的讲解如何创建 Yii2 项目，和其中需要注意的地方。
---
## 前言
### Yii
  [Yii](http://www.yiiframework.com/) 是一个高性能的，适用于开发 WEB2.0 应用的 PHP 框架。
  Yii 自带了丰富的功能 ，包括 MVC，DAO/ActiveRecord，I18N/L10N，缓存，身份验证和基于角色的访问控制，脚手架，测试等，可显著缩短开发时间。
## 创建
### 前期准备
  * [PHP运行环境](/2016-09-30/Install-Nginx-PHP7-MySQL-on-Ubuntu16-04.html)
  * Composer 环境
    - Mac OS X
      
          ```bash
         brew install composer
          ```
    - Ubuntu
      
      ```bash
        # 下载
        curl -sS https://getcomposer.org/installer | php -d detect_unicode=Off 
        # 全局
        mv composer.phar /usr/local/bin/composer 
        # 权限
        chmod a+x /usr/local/bin/composer
        # 更新
        composer self-update
      ```
### 安装
1. #### Composer 安装
     安装composer-asset-plugin插件
      ```bash
      # 切换国内镜像(http://pkg.phpcomposer.com/)
      composer config -g repo.packagist composer https://packagist.phpcomposer.com
      
      composer global require "fxp/composer-asset-plugin:^1.2.0"
      ```
      安装基础版
      ```bash
      composer create-project yiisoft/yii2-app-basic xxx
      ```
      安装高级版
      ```bash
      composer create-project yiisoft/yii2-app-advanced xxx
      ```
2. #### 下载安装
      [基础班](https://github.com/yiisoft/yii2/releases/download/2.0.10/yii-basic-app-2.0.10.tgz)
      [高级版](https://github.com/yiisoft/yii2/releases/download/2.0.10/yii-advanced-app-2.0.10.tgz)

## Tips
  ```bash
  The zip extension and unzip command are both missing, skipping.
  ```
  解决方案
  ```bash
  apt-get install php7.0-zip
  ```
  ---
  ```bash
  codeception/base 2.2.3 requires phpunit/phpunit >4.8.20 <5.5 -> satisfiable by phpunit/phpunit
  phpunit/phpunit 5.6.2 requires ext-dom * -> the requested PHP extension dom is missing from your system.
  ```
  解决方案
  ```bash
  apt-get install php7.0-xml
  ```