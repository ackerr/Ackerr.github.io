---
title: 初识Flask
date: 2018-08-12 15:25:56
tags:
    - Python
    - Flask
---
> 本文简单的介绍了在开始编写flask 项目前的一些操作
1.安装环境与依赖
2.最小的flask
<!-- more -->
#### 安装python3（具体的版本，看需求而定）

一般情况下,每个不同的项目使用自己的虚拟环境,一个好处就是可以选择使用的Python版本

#### 安装Python 虚拟环境
```bash
pip install virtualenv
```
#### 激活虚拟环境

- Linux/ Mac ox

```bash
    source env/bin/activate
```
- Windows

```bash
    source env/Scripts/activate
```
>为省方便，可以在本地添加一条alias ==sb=source env/bin/activate==,具体加alias的方式可以自行搜索

#### 安装环境并进入环境后，接着就可以安装flask框架了
```bash
    pip install flask
```
> pycharm 中也有一步到位，傻瓜式安装框架，并且会生成一些文件，例如app.py

#### 新建一个文件hello.py，内容如下，最小的应用就已经写好了

```python
from flask import Flask  # 导入Flask类
app = Flask(__name__)  # 新建一个类对象

@app.route('/')  # flask 的路由系统
def hello_world():  # 具体的逻辑
    return 'Hello World!'

if __name__ == '__main__':
    app.run(debug=True)  # 运行方法,debug=True时，适合本地调试模式
```

