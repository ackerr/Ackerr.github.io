---
title: Pipenv - Python包管理工具
date: 2018-11-04 23:07:09
tags:
    - Python
categories:
    - 工具
---
> 本文简单介绍下最近接触的 pipenv (Python 包管理工具之一)，随带回顾一下`pip + virtualenv`。



Pipenv  是 <a href='Python.org'>python 官网</a> 最新推荐的包管理工具，主要功能跟使用`pip + virtualenv`差不多，不过官网推荐使用，肯定是有其更优秀的地方，总结一下，pipenv解决了一下几个问题：

- 不再需要手动创建虚拟环境
- 使用Pipfile 与 Pipfile.lock 替代 requirements.txt 文件
- 使用hash校验依赖
- 可查看图形化依赖关系



## 安装与使用

```bash
pip install --user --upgrade pipenv
```

> 前提是已经装好Python 与 pip

```bash
# 如果安装后，无法使用pipenv命令，需要多做一步操作，将Python路径加到环境变量中

vim ~/.profile

export PATH=$PATH:<Python路径>
```

如果不知道python 路径可以输入一下命令查看

```bash
python -m site --user-base
```

接下来给项目安装pipenv，执行

```bash
pipenv install
```

执行结果如下图

![执行结果](http://pdc0iq8f2.bkt.clouddn.com/pipenv.png)

观察执行过程，可发现该命令做了几步。

- 第一步：先给项目自动创建了一个虚拟环境，等同于`virtualenv env`，不过pipenv 将所有的虚拟环境都放置在一处地方，通过`pipenv --venv`查看项目的虚拟环境位置

- 第二步：生成了两个文件`Pipfile and Pipfile.lock`


#### Pipfile

```markdown
[[source]]    # pip源，可将默认源换成其他源，如下面的豆瓣源
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"

[[source]]
url = "https://pypi.douban.com/simple"
verify_ssl = true
name = "douban"

[packages]   # 依赖与版本
celery = "==4.2.1"

[dev-packages]
flake8 = ">=3.6.0"

[requires]    # python版本
python_version = "3.6"
```

#### Pipfile.lock

根据 `Pipfile` 和当前环境自动生成的 JSON 格式的依赖文件，**任何情况下都不要手动修改该文件！**


安装完毕后，首先**激活虚拟环境**

```bash
pipenv shell  # 本质上与 source env/bin/activate 相同

exit  # 退出虚拟环境,类比  deactivate
```

接下来就是使用`pipenv `来操作我们的项目依赖

```bash
pipenv install requests
```

查看 Pipfile 文件内容，会发现这个文件记录了依赖，值得一提的是pipenv 可以分类依赖，观察`Pipfile`可知分为`packages/dev-packages` ，比如编码规范类型的依赖就可以放到dev-packages中，可以使用如下命令，从而分类依赖

```bash
pipenv install ipython --dev
```

> pipenv默认安装了flake8，可以执行`pipenv check`试试哦


在`requirements.txt`中，个人认为最直观的问题就是会把第三方库的依赖也包括进去, 这样其实只装一样东西，可能

`pip freeze >requirements.txt`后会出现好多依赖，而`Pipfile`并不会，如果想**查看依赖关系**

```bash
pipenv graph
```

#### 吐槽向

使用pipenv后，一些shell 命令 如`python manage.py test` 需要变成 `pipenv run python manage.py test`，

安装依赖时，会触发lock , 感觉有点久呀~，所以看了一下可以将`install`与`lock`操作分开，或者换pip源

```bash
pipenv install xxx  --skip-lock

# 后续再操作
pipenv lock
```

#### 总结一下pipenv的基本命令

- `pipenv --where`:  项目的根目录
- `pipenv --venv`: 项目的虚拟环境路径
- `pipenv install`:  安装`Pipfile`中的依赖，没有则生成`Pipfile`
- `pipenv install --dev`:  安装`Pipfile`中dev-pakcages的依赖
- `pipenv install xx --skip-lock`: 安装依赖，但不进行lock
- `pipenv uninstall`: 卸载所有依赖
- `pipenv shell`: 进入虚拟环境
- `pipenv lock`: 确认依赖已安装，生成相应版本的`Pipfile.lock`
- `pipenv check`: 检查代码规范
- `pipenv graph`: 显示依赖关系
