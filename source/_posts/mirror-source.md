---
title: 常用依赖镜像源 
date: 2020-11-29 19:50:16
categories:
    - 工具
---

> 一句话：从右边几个源中寻找对应软件即可 [清华源](https://mirror.tuna.tsinghua.edu.cn/help/)   [中科大源](https://mirrors.ustc.edu.cn/help/)  [阿里源](https://developer.aliyun.com/mirror/)   [淘宝源](https://npm.taobao.org/mirrors) 。当然如果能科学上网可忽略，不过大多数时候使用国内镜像源会更快一些。

趁这波换电脑，顺便整理了一番个人日常开发中需要替换的国内镜像源。

## Python

**pip**

以下命令会添加配置至`$HOME/.config/pip/pip.conf`

```shell
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pip -U
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

**pyenv**

修改v变量，一键安装对应Python版本

```shell
v=3.8.3;wget https://npm.taobao.org/mirrors/python/$v/Python-$v.tar.xz -P $(pyenv root)/cache/;pyenv install $v
```

## Golang

**go module**

```shell
export GO111MODULE=on
export GOPROXY='https://goproxy.cn'
```

## Node.js

**npm**

```shell
npm config set registry https://registry.npm.taobao.org
```

**yarn**

```shell
yarn config set registry https://registry.npm.taobao.org/
```

---

## Arch

**pacman**

覆盖原有`/etc/pacman.d/mirrorlist`，可以先备份原有文件

```shell
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.aliyun.com/archlinux/$repo/os/$arch
```

**archlinuxcn**

添加至原`/etc/pacman.conf`末尾

```shell
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

**AUR**

```shell
yay --aururl "https://aur.tuna.tsinghua.edu.cn" --save
```

## Ubuntu

**20.04LTS**

覆盖原有`/etc/apt/srouces.list`

```shell
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
```

`18.04LTS`版本将所有 focal 替换成 bionic 即可


## Alpine

```shell
sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
```

---

## Homebrew

**Core**

```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
```

**Cask**

```shell
cd "$(brew --repo)"/Library/Taps/homebrew/homebrew-cask
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git
```

---

不定时更新中...
