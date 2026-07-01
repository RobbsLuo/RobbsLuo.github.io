---
title: GEM Notes
date: 2016-10-12 15:44:28
tags:
  -  Software
  -  GEM
  -  Ruby
categories:
  -  Software
description: A collection of commonly used GEM-related commands.
lang: en
---
## Introduction
This article records commands that frequently come up during day-to-day use.

## Reference

### Switching the mirror
Because of network constraints in mainland China, the very first thing we often do before using many tools is switch to a domestic mirror. [Taobao mirror](https://ruby.taobao.org/ 'Taobao mirror')
```bash
gem sources --remove https://rubygems.org/
gem sources -a https://ruby.taobao.org/

gem sources -l
```
```bash
*** CURRENT SOURCES ***

https://ruby.taobao.org
# Make sure only ruby.taobao.org remains
```

### Common commands
```bash
gem -v                                 # Check the version of RubyGems

gem help                               # Show RubyGems help
gem help example                       # List RubyGems command examples

gem install [gemname]                  # Install the specified gem. It first looks locally, then falls back to the remote gem source if the gem is not found locally.
gem install -l [gemname]               # Install the gem from the local source only
gem install -r [gemname]               # Install the gem from the remote source only
gem install [gemname] --version=[ver]  # Install a specific version of the gem

gem uninstall [gemname]                # Remove the specified gem. Note: this removes all installed versions.
gem uninstall [gemname] --version=[ver] # Remove a specific version of the gem

gem update --system                     # Update RubyGems itself
gem update [gemname]                    # Update all or a specified installed gem

gem list                                # List all gems installed locally # Show RubyGems help

gem list | cut -d" " -f1 | xargs gem uninstall –aIx  # Remove all gems in the system
# -I Ignore dependency requirements while uninstalling
# -a Uninstall all matching versions
# -x Uninstall applicable executables without confirmation

gem cleanup                             # Remove all old versions of each package, keeping only the latest  -d preview what will be removed   -v

gem environment                         # Inspect the gem environment
```
