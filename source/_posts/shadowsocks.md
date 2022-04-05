---
title: 科学上网
date: 2018-11-27 18:05:09
tags:
    - shadowsocks
    
categories: 
    - 工具
---
> 这篇文件简单讲述有志青年如何从`村通网`跨越重重障碍到`科学上网`的经历



**前提条件：需要一个国外的vps 本地设SSH Key**

在某人的忽悠下购买了`DigitalOcean`的`Ubuntu`服务器（~~~此处不是广告~~）

通过 ssh 连上服务器 

```bash
# 假设服务器 公网IP 127.0.0.1
ssh root@127.0.0.1  

# 输入密码后就连上了服务器
```

<!-- more -->

## 搭建shadowsocks

### 安装环境

```bash
# Ubuntu:
apt-get install python-pip

pip install shadowsocks
```

> 注：这里遇到了几个坑

- 在安装pip时遇到`ImportError:cannot import name main`的报错 

![解决方法](/images/shadowsocks1.png)

- pip install 时遇到`locale.Error: unsupported locale setting`报错 

![解决办法](/images/shadowsocks2.png)



### 配置文件

**vim /etc/shadowsocks.json**

```bash
{
    "server":"0.0.0.0"
    "server_post":9898,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"password",
    "timeout":300,
    "method":"aes-256-cfb"
}
```



### 启动服务

```bash
# start
ssserver -c /etc/shadowsocks.json -d start

# stop
ssserver -c /etc/shadowsocks.json -d stop
```



### 打开服务相应端口

```bash
firewall-cmd --zone=public --add-port=9898/tcp --permanent

systemctl restart firewalld.service
```
> **到此就可以连接使用了**

### 设置定时任务脚本

可参考：
<a href='https://shadowsocks.be/6.html'>Shadowsocks定时任务脚本</a>  <a href='https://teddysun.com/489.html'>Shadowsocks bbr脚本</a>


## 简化操作

为了方便操作，可以添加一些配置

```bash
vim ~/.ssh/config

# config 中添加
Host alias    # 别名,可修改
    HostName 127.0.0.1  # 公网IP
    User     root       # 登录用户
```

添加配置之后， 就可以直接`ssh alias` 输入密码就登上服务器了

每次登录都要输入密码，为了更方便，可以免密登录，将本地的公钥上传到服务器

```bash
scp ~/.ssh/id_rsa.pub  root@127.0.0.1:~
```

登上服务器， 将公钥写入`authorized_keys`文件中，就可以免密登录啦

```bash
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
```

