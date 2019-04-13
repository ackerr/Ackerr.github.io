---
title: 搭建日志监控服务 Sentry
date: 2019-04-13 19:10:47
tags:
    - sentry
    - docker
categories:
    - 搭建系列

---

> 不写BUG的码农不是好码农！

Sentry官方提供了两种搭建方式（Python/Docker），这里我们使用docker进行搭建。

### 前情提要

如果机器上没有 `git`, `docker`以及 `docker-compose`， 先进行安装，**本文以 `ubuntu16.04 `为例**

```bash
apt-get install docker.io git
```

如果本地有`python`环境，推荐使用`pip`进行安装`docker-compose`

```bash
pip install docker-compose
```

> 当然也可以使用 `apt-get`，不过下载的版本可能不对，导致`docker`与`docker-compose.yml`中的版本不匹配
> [docker 与 docker-compose版本关系](https://docs.docker.com/compose/compose-file/compose-versioning/)

### 服务搭建

[Sentry](https://github.com/getsentry/onpremise)提供了docker搭建的`docker-compose.yml`以及对应的`Dockerfile`，首先拉取官方提供的docker文件。

```bash
git clone https://github.com/getsentry/onpremise.git
```
<!-- more -->

剩余的操作如同`README.md`中的步骤一致

- 进入目录`cd onpremise`

- 创建对应的`volumn`文件夹，用户存放数据

```bash
docker volume create --name=sentry-data && docker volume create --name=sentry-postgres
```

- 创建环境变量配置文件

```bash
cp -n .env.example .env
```

- 构建docker镜像

```
docker-compose build
```

- 生成secret key，将其添加到配置文件`.env`中 (生成的secret key 会在最后一行)

```bash
docker-compose run --rm web config generate-secret-key
```

- 更新配置，其中会进行`database migrate`，最后会提示创建一个登录账号

```bash
docker-compose run --rm web upgrade
```

- 最后执行命令， 后台运行`sentry server`

```bash
docker-compose up -d
```

接着我们浏览器访问 `localhost:9000` 或者 `server ip:9000`，访问至登录页面即成功。

### 邮件配置

Sentry中提供了邮件邀请用户，从而进行团队协作，不过需要稍加配置

`vim config.yml`, 可以看到有如下配置

```yml
###############
# Mail Server #
###############

# mail.backend: 'smtp'  # Use dummy if you want to disable email entirely
# mail.host: 'localhost'
# mail.port: 25
# mail.username: ''
# mail.password: ''
# mail.use-tls: false
```

根据自己的需求，更改配置，这里以`qq邮箱`为例：

```yml
mail.backend: 'smtp'
mail.host: 'smtp.qq.com'
mail.port: 587
mail.username: '1352252181@qq.com'  # 发送邮箱地址
mail.password: '*****'              # 这里是邮箱授权码不是邮箱密码
mail.use-tls: true
```

> sentry 只支持tls而不是ssh，所以需要port不是465

完成后，重启服务即可

```bash
docker-compose down   # 关闭相关docker container
docke-compose up -d   # 后台运行server
```

### 服务部署

这里简单讲下通过`Nginx` 绑定域名，部署`sentry`服务，并使用certbot 使服务支持`https`

- 查看本地有无nginx以及cerbot，如果没有先安装

```bash
# nginx
apt-get install nginx

# certbot
apt-get install software-properties-common
add-apt-repository ppa:certbot/certbot
apt-get update
apt-get install python-certbot-nginx
```

- 新建Nginx配置文件

```bash
vim /etc/nginx/conf.d/sentry.conf
```

配置如下：

```yml
server{
    listen 80;
    server_name xxxx;

    location / {
        proxy_set_header Host $host;
        proxy_pass http://127.0.0.1:9000/;
    }
}
```

- 支持服务至`Https`

```bash
certbot --nginx
```

**过程中会有两次选项，第一个会让选择升级的域名，也就是更改相应的Nginx配置文件，第二次则**

再次打开`Nginx`配置文件，发现加了许多配置

> `certbot`的方便之处，免费签发证书，一键支持https

```yml
server{
    server_name xxxx;

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_connect_timeout 180;
        proxy_pass http://127.0.0.1:9000/;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/xxxx/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/xxxx/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server{
    listen 80;
    server_name xxxx;
    return 301 https://xxxx$request_uri;
}
```

- 设置定时任务，定时更新HTTPS证书

```
vim /etc/crontab

# 加入下面这句
0 3 * * * certbot renew
```

- `service nginx restart`， 重启服务

访问域名，成功登陆后，到此Sentry服务的搭建完成。

![成果图](https://picture.wzmmmmj.com/sentry-web.jpg)


> Sentry还支持[一些插件]()，可以根据需要进行配置，例如jira ….
> 不过如果是个人或少数人，懒得折腾，更适合使用sentry官方提供的服务 [sentry.io](https://sentry.io/)， 只需要注册即可使用。

> 参考文档：[sentry官方文档](https://docs.sentry.io/server/installation/docker/) ，[onpremise项目地址](https://github.com/getsentry/onpremise)

