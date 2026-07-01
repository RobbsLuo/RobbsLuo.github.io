---
title: Deploying a Yii2 Project with Mina
date: 2016-10-25 23:03:20
tags:
  - Mina
  - Yii2
  - PHP
  - Deploy
categories:
  - Develop
description: Quickly deploy a Yii2 project using Mina.
lang: en
---
## Introduction
### Mina
  Really fast deployer and server automation tool.
  Mina is a fast deployment tool. Its deployment scripts are simple, it is highly extensible, deployment is fast (only a single SSH connection), and the deployment output is concise.

## Deployment
### Prerequisites
  * A [Yii2](/2016-10-25/Create-Yii2-Project.html 'Creating a Yii2 Project') project repository
  * A VPS with a [basic environment](/2016-09-30/Install-Nginx-PHP7-MySQL-on-Ubuntu16-04.html) installed
  * A development machine with Ruby installed
### Install Mina
  ```bash
  gem install mina
  ```
### Initialize the configuration
  ```bash
  mina init
  
  # Generated config file
  # ├── config
  # │   └── deploy.rb
  ```
 ### Edit the configuration file
   ```bash
   vim deploy.rb
   ```
   My configuration:
   ```bash
    require 'mina/git'  # git support
    
    # Basic settings:
    #   domain       - The hostname to SSH to.
    #   deploy_to    - Path to deploy into.
    #   repository   - Git repo to clone from. (needed by mina/git)
    #   branch       - Branch name to deploy. (needed by mina/git)

    set :deploy_to, '/var/www/html/test'             # Directory on the VPS used for deployment
    set :repository, 'git@github.com:xxx/test.git'   # GitHub repository URL
    set :branch, 'develop'                           # Branch used for deployment

    set :keep_releases, 4              # Number of releases to keep

    # Manually create these paths in shared/ (eg: shared/config/database.yml) in your server.
    # They will be linked in the 'deploy:link_shared_paths' step.
    set :shared_paths, ['vendor', 'runtime', 'web/assets']    # Shared directories

    # Optional settings:
    #   set :user, 'foobar'    # Username in the server to SSH to.
    #   set :port, '30000'     # SSH port number.
    #   set :forward_agent, true     # SSH forward_agent.
    
    set :user, 'ubuntu'                              # Username for logging into the VPS
    set :domain, 'x.x.x.x'                           # IP address of the VPS

    # This task is the environment that is loaded for most commands, such as
    # `mina deploy` or `mina rake`.
    task :environment do
    end

    # Put any custom mkdir's in here for when `mina setup` is ran.
    # For Rails apps, we'll make some of the shared paths that are shared between
    # all releases.
    task :setup => :environment do
      # Project initialization: create shared folders and install Yii2 Composer support
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
        invoke :'git:clone'                          # Update code
        invoke :'deploy:link_shared_paths'           # Link shared directories
        queue 'chmod -R 755 yii'                     # Permissions 
        queue 'composer install'                     # Install Composer packages
        queue './yii migrate'                        # Database migration
        quequ 'rm -rf runtime/cache/*'               # Clear the cache
        quequ 'service nginx restart'                # Restart Nginx
        quequ 'service php7.0-fpm restart'           # Restart PHP
        invoke :'deploy:cleanup'                     # Clean up redundant releases
      end
    end

    # Used to roll back to the previous version
    desc "Rollback to previous verison."
    task :rollback => :environment do
      queue %[echo "----> Start to rollback"]
      queue %[if [ $(ls #{deploy_to}/releases | wc -l) -gt 1 ]; then echo "---->Relink to previos release" && unlink #{deploy_to}/current && ln -s #{deploy_to}/releases/"$(ls #{deploy_to}/releases | tail -2 | head -1)" #{deploy_to}/current && echo "Remove old releases" && rm -rf #{deploy_to}/releases/"$(ls #{deploy_to}/releases | tail -1)" && echo "$(ls #{deploy_to}/releases | tail -1)" > #{deploy_to}/last_version && echo "Done. Rollback to v$(cat #{deploy_to}/last_version)" ; else echo "No more release to rollback" ; fi]
    end
   ```
 ### Initialize the deployment environment 
 ```bash
 mina setup
 ```
 After running this, Mina creates its specific release directory structure in the configured deployment directory and executes the `setup` section of the configuration.
 
### Modify Nginx
  ```bash
  server {
      listen       80;
      server_name  test.com;
      index index.html index.htm index.php;
      root /var/www/html/test/current;    # Append current to the deployment directory
  ```
 ### Start deploying
 Run the following command every time you need to deploy.
   ```bash
    mina deploy
   ```
   
## Tips
 * Add the development machine used for deployment to the VPS passwordless login configuration.
 * Add the VPS's SSH key to the Deploy keys of the GitHub repository.
