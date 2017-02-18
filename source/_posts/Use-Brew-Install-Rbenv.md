---
title: Mac OS 下安装 rbenv
date: 2016-12-26 16:41:17
tags:
- rbenv
- brew
- ruby
categories: Software
description: rbenv 和 rvm 用来管理多个版本的 ruby 安装和使用。
---

## 简介

目前Mac OS 下面有两种Ruby 版本的工具，分别是 RVM 和 rbenv，本文主要介绍如何安装和使用 rbenv，并通过它来管理 Ruby 环境

## 安装

### 安装

```
brew install rbenv
brew install ruby-build
brew install rbenv-gemset
brew install rbenv-gem-rehash
```

### 初始化

```
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc

# 如果使用的是 Zsh  
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(rbenv init -)"' >> ~/.zshrc
```

### 安装Ruby

```shell
# 查看可用版本
rbenv install --list

# 安装2.4.0
rbenv install 2.4.0

# 查看已安装版本
rbenv versions
#  system
#   2.1.5
#   2.2.1
# * 2.2.4 (set by /Users/Robbs/.rbenv/version)

# 设置全局版本
rbenv global 2.4.0

# 设置本地版本
rbenv local 2.4.0

# 取消设置
rbenv local --unset

# 设置当前终端版本
rbenv shell 2.4.0

# 使用系统Ruby
rbenv global system

# 每当切换ruby版本和执行bundle install之后必须执行这个命令
rbenv rehash

# 卸载Ruby
rbenv uninstall 2.4.0
```

### 切换镜像

```shell
# rbenv-china-mirror
git clone https://github.com/andorchen/rbenv-china-mirror.git ~/.rbenv/plugins/rbenv-china-mirror
```