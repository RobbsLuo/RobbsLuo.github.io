---
title: GEM 笔记
date: 2016-10-12 15:44:28
tags:
  -  Software
  -  GEM
  -  Ruby
categories:
  -  Software
description: 记录一些比较常用的GEM相关的命令。
---
## 前言
本文记录了在使用过程中经常出现的一些命令

## 记录

### 更换镜像
由于国内网络问题，在使用很多东西前我们第一件事情就是换镜像。[淘宝镜像](https://ruby.taobao.org/ '淘宝镜像')
```bash
gem sources --remove https://rubygems.org/
gem sources -a https://ruby.taobao.org/

gem sources -l
```
```bash
*** CURRENT SOURCES ***

https://ruby.taobao.org
# 请确保只有 ruby.taobao.org
```

### 常用命令
```bash
gem -v                                 # 查看RubyGems软件的版本

gem help                               # 显示RubyGem使用帮助
gem help example                       # 列出RubyGem命令一些使用范例

gem install [gemname]                  # 安装指定gem包，程序先从本机查找gem包并安装，如果本地没有，则从远程gem安装。
gem install -l [gemname]               # 仅从本机安装gem包
gem install -r [gemname]               # 仅从远程安装gem包
gem install [gemname] --version=[ver]  # 安装指定版本的gem包

gem uninstall [gemname]                # 删除指定的gem包，注意此命令将删除所有已安装的版本
gem uninstall [gemname] --version=[ver] # 删除某指定版本gem

gem update --system                     # 更新升级RubyGems软件自身
gem update [gemname]                    # 更新所有|指定已安装的gem包

gem list                                # 查看本机已安装的所有gem包 # 显示RubyGem使用帮助

gem list | cut -d" " -f1 | xargs gem uninstall –aIx  # 删除系统中的所有gems
# -I Ignore dependency requirements while uninstalling
# -a Uninstall all matching versions
# -x Uninstall applicable executables without confirmation

gem cleanup                             # 清除所有包旧版本，保留最新版本  -d 查看那些会被删除   -v

gem environment                         # 查看gem的环境
```