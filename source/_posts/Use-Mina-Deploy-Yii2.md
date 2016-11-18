---
title: 使用 Mina 部署 Yii2 项目
date: 2016-10-25 23:03:20
tags:
  - Mina
  - Yii2
  - PHP
  - Deploy
categories:
  - Develop
description: 利用 Mina 快速的发布 Yii2 项目。
---
## 前言
### Mina
  Really fast deployer and server automation tool.
  Mina 是一款快速部署工具。它部署脚本简单，扩展性强，部署速度快 (只有一次 SSH 连接)，部署信息简单。
 
## 部署
### 前期准备
  * 一个 [Yii2](/2016-10-25/Create-Yii2-Project.html '创建 Yii2 项目') 的项目仓库
  * 一台服务器安装了[基础环境](/2016-09-30/Install-Nginx-PHP7-MySQL-on-Ubuntu16-04.html)的VPS
  * 一台安装有 Ruby 环境的开发机
### 安装Mina
  ```bash
  gem install mina
  ```
### 初始化配置
  ```bash
  mina init
  
  # 生成配置文件
  # ├── config
  # │   └── deploy.rb
  ```
 ### 修改配置文件
   ```bash
   vim deploy.rb
   ```
   我的配置
   ```bash
    require 'mina/git'  # git 支持
    
    # Basic settings:
    #   domain       - The hostname to SSH to.
    #   deploy_to    - Path to deploy into.
    #   repository   - Git repo to clone from. (needed by mina/git)
    #   branch       - Branch name to deploy. (needed by mina/git)

    set :deploy_to, '/var/www/html/test'             # VPS 上用来发布的目录
    set :repository, 'git@github.com:xxx/test.git'   # github 仓库地址
    set :branch, 'develop'                           # 用来发布的分支

    set :keep_releases, 4              # 保留的发布版本数

    # Manually create these paths in shared/ (eg: shared/config/database.yml) in your server.
    # They will be linked in the 'deploy:link_shared_paths' step.
    set :shared_paths, ['vendor', 'runtime', 'web/assets']    # 设置共享目录

    # Optional settings:
    #   set :user, 'foobar'    # Username in the server to SSH to.
    #   set :port, '30000'     # SSH port number.
    #   set :forward_agent, true     # SSH forward_agent.
    
    set :user, 'ubuntu'                              # 登录 VPS 的用户名
    set :domain, 'x.x.x.x'                           # VPS 的 IP 地址

    # This task is the environment that is loaded for most commands, such as
    # `mina deploy` or `mina rake`.
    task :environment do
    end

    # Put any custom mkdir's in here for when `mina setup` is ran.
    # For Rails apps, we'll make some of the shared paths that are shared between
    # all releases.
    task :setup => :environment do
      # 项目初始化，生成共享文件夹，安装 Yii2 Composer 支持
      queue! %[mkdir -p "#{deploy_to}/#{shared_path}/runtime"]
      queue! %[mkdir -p "#{deploy_to}/#{shared_path}/vendor"]
      queue! %[mkdir -p "#{deploy_to}/#{shared_path}/web/assets"]
      queue! %[chmod -R 777 "#{deploy_to}/#{shared_path}/runtime"]
      queue! %[chmod -R 777 "#{deploy_to}/#{shared_path}/web/assets"]
      queue 'composer global require "fxp/composer-asset-plugin:^1.2.0"'
    end

    desc "Deploys the current version to the server."
    task :deploy => :environment do
      to :before_hook do
        # Put things to run locally before ssh
      end
      deploy do
        # Put things that will set up an empty directory into a fully set-up
        # instance of your project.
        invoke :'git:clone'                          # 更新代码
        invoke :'deploy:link_shared_paths'           # 链接共享目录
        queue 'chmod -R 755 yii'                     # 权限 
        queue 'composer install'                     # 安装Composer包
        queue './yii migrate'                        # 数据库迁移
        quequ 'rm -rf runtime/cache/*'               # 清空缓存
        quequ 'service nginx restart'                # 重启 Nginx
        quequ 'service php7.0-fpm restart'           # 重启 PHP
        invoke :'deploy:cleanup'                     # 清除多余的发布
      end
    end

    # 用于回滚到上一个版本
    desc "Rollback to previous verison."
    task :rollback => :environment do
      queue %[echo "----> Start to rollback"]
      queue %[if [ $(ls #{deploy_to}/releases | wc -l) -gt 1 ]; then echo "---->Relink to previos release" && unlink #{deploy_to}/current && ln -s #{deploy_to}/releases/"$(ls #{deploy_to}/releases | tail -2 | head -1)" #{deploy_to}/current && echo "Remove old releases" && rm -rf #{deploy_to}/releases/"$(ls #{deploy_to}/releases | tail -1)" && echo "$(ls #{deploy_to}/releases | tail -1)" > #{deploy_to}/last_version && echo "Done. Rollback to v$(cat #{deploy_to}/last_version)" ; else echo "No more release to rollback" ; fi]
    end
   ```
 ### 初始化部署环境 
 ```bash
 mina setup
 ```
 运行后会在配置的发布目录中创建特有的发布目录结构，并执行配置中 `setup` 部分。
 
### 修改 Nginx
  ```bash
  server {
      listen       80;
      server_name  test.com;
      index index.html index.htm index.php;
      root /var/www/html/test/current;    # 在发布目录上增加 current
  ```
 ###  开始部署
 在每一次需要部署时运行以下命令
   ```bash
    mina deploy
   ```
   
## Tips
 * 将发布的开发机加入到 VPS 的免密码登录配置
 * 将 VPS 的 SSH Key 加入到 github 仓库的  Deploy keys 内