---
title: Creating a Yii2 Project
date: 2016-10-25 22:07:50
tags:
  - PHP
  - Yii2
categories:
  -  Develop
description: A brief guide to creating a Yii2 project, along with a few things to watch out for.
lang: en
---
## Introduction
### Yii
  [Yii](http://www.yiiframework.com/) is a high-performance PHP framework suitable for developing Web 2.0 applications.
  Yii ships with a rich set of features, including MVC, DAO/ActiveRecord, I18N/L10N, caching, authentication and role-based access control, scaffolding, testing, and more, which can significantly shorten development time.
## Creation
### Prerequisites
  * [PHP runtime environment](/2016-09-30/Install-Nginx-PHP7-MySQL-on-Ubuntu16-04.html)
  * Composer environment
    - Mac OS X
      
          ```bash
         brew install composer
          ```
    - Ubuntu
      
      ```bash
        # Download
        curl -sS https://getcomposer.org/installer | php -d detect_unicode=Off 
        # Make it global
        mv composer.phar /usr/local/bin/composer 
        # Permissions
        chmod a+x /usr/local/bin/composer
        # Update
        composer self-update
      ```
### Installation
1. #### Install via Composer
     Install the composer-asset-plugin.
      ```bash
      # Switch to the China mirror (http://pkg.phpcomposer.com/)
      composer config -g repo.packagist composer https://packagist.phpcomposer.com
      
      composer global require "fxp/composer-asset-plugin:^1.2.0"
      ```
      Install the basic edition.
      ```bash
      composer create-project yiisoft/yii2-app-basic xxx
      ```
      Install the advanced edition.
      ```bash
      composer create-project yiisoft/yii2-app-advanced xxx
      ```
2. #### Download and install
      [Basic edition](https://github.com/yiisoft/yii2/releases/download/2.0.10/yii-basic-app-2.0.10.tgz)
      [Advanced edition](https://github.com/yiisoft/yii2/releases/download/2.0.10/yii-advanced-app-2.0.10.tgz)

## Tips
  ```bash
  The zip extension and unzip command are both missing, skipping.
  ```
  Solution
  ```bash
  apt-get install php7.0-zip
  ```
  ---
  ```bash
  codeception/base 2.2.3 requires phpunit/phpunit >4.8.20 <5.5 -> satisfiable by phpunit/phpunit
  phpunit/phpunit 5.6.2 requires ext-dom * -> the requested PHP extension dom is missing from your system.
  ```
  Solution
  ```bash
  apt-get install php7.0-xml
  ```
