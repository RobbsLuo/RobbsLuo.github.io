---
title: Installing rbenv on Mac OS
date: 2016-12-26 16:41:17
lang: en
tags:
- rbenv
- brew
- ruby
categories: Software
description: rbenv and rvm are used to manage the installation and usage of multiple versions of Ruby.
---

## Introduction

There are currently two tools for managing Ruby versions on Mac OS: RVM and rbenv. This article mainly covers how to install and use rbenv, and how to manage your Ruby environment through it.

## Installation

### Installation

```
brew install rbenv
brew install ruby-build
brew install rbenv-gemset
brew install rbenv-gem-rehash
```

### Initialization

```
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc

# If you are using Zsh
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(rbenv init -)"' >> ~/.zshrc
```

### Installing Ruby

```shell
# List available versions
rbenv install --list

# Install 2.4.0
rbenv install 2.4.0

# Show installed versions
rbenv versions
#  system
#   2.1.5
#   2.2.1
# * 2.2.4 (set by /Users/Robbs/.rbenv/version)

# Set the global version
rbenv global 2.4.0

# Set the local version
rbenv local 2.4.0

# Unset the local version
rbenv local --unset

# Set the version for the current shell
rbenv shell 2.4.0

# Use the system Ruby
rbenv global system

# You must run this command after switching Ruby versions and after running bundle install
rbenv rehash

# Uninstall a Ruby version
rbenv uninstall 2.4.0
```

### Switching the mirror

```shell
# rbenv-china-mirror
git clone https://github.com/andorchen/rbenv-china-mirror.git ~/.rbenv/plugins/rbenv-china-mirror
```
