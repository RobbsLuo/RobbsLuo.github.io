---
title: Mac OS 基础软件
date: 2016-12-26 16:39:02
tags:
  - Software
  - OSX
categories: Software
description: 主要介绍在 Mac OS 下使用的基本的软件。
---

## 简介

主要介绍在 Mac OS 下使用的基本的软件。

## 软件

### Command Line Tool

```
xcode-select --install
```

### Homebrew

[Homebrew](http://brew.sh/,)

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

### Oh My Zsh

[Oh My Zsh](https://github.com/robbyrussell/oh-my-zsh)

```
brew install zsh

sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

### Prezto

[Prezto](https://github.com/sorin-ionescu/prezto)

```
# Launch Zsh:
zsh

#Clone the repository:
git clone --recursive https://github.com/sorin-ionescu/prezto.git "${ZDOTDIR:-$HOME}/.zprezto"

#Create a new Zsh configuration by copying the Zsh configuration files provided:
setopt EXTENDED_GLOB
for rcfile in "${ZDOTDIR:-$HOME}"/.zprezto/runcoms/^README.md(.N); do
  ln -s "$rcfile" "${ZDOTDIR:-$HOME}/.${rcfile:t}"
done

# Set Zsh as your default shell:
chsh -s /bin/zsh
```

### Brew Cask

[Homebrew Cask](https://caskroom.github.io/)

```
brew tap phinze/cask
brew install brew-cask
```

### TotalTerminal

```
brew cask install totalterminal
```

### LaunchRocket

```
brew cask install launchrocket
```

### shadowssocks

```
https://github.com/shadowsocks/shadowsocks-iOS/releases
```

### alfred 3

```
brew cask install alfred
```

### moom

```
brew cask install moom
```