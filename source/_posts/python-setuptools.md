---
title: python 打包发布
date: 2018-08-26 19:29:45
tags: 
    - Python
---
最近同事因为一个需求需要将代码打包上传到内网中，跟着小学了一手， 所以这里将打包发布方法记录一下

> 其实最重要的就是一个setup.py 文件
<!-- more -->
### 打包

#### 1.distutils(标准库)

python3中可直接使用标准库 distutils, 代码如下

```python
from distutils.core import setup


setup(
    name='bot',  # 包名称
    version='0.0.1',  # 工程版本号
    description='微信机器人',  # 对包的描述
    url='https://github.com/zzzzzzmj',  # 包地址
    author='ZmJ',  # 作者信息
    author='xxx'  # 作者邮箱
    license='',  # 许可证信息
    install_requires=[
        'wxpy>=0.3.9.8',
    ]
)
```

其中`install_requires`中的参数是包中的其他依赖，会在pip install 时，一同下载，其余信息都是一些个人或对包的信息介绍，更多参数可以查看源码。

简单的说，打包分为以下几步

1. 创建package同名文件夹
2. 将package放到文件夹中，同时新建文件setup.py
3. 编辑setup.py文件
4. 运行命令`python setup.py sdist`生成一个`dist/ xxx.tar.gz`的压缩包

#### 2.setuptools(第三方库)

> 一句话就是优化版的distutils,不过需要先安装setuptools库

主要优化了distutils几个功能，其余都差不多

- 不会新建MANIFEST文件，自动包含了所有相关的data文件
- 自动包含所有的包，distutils需要在参数中列出

### 发布

通过以下几步上传库到PyPI

1. 注册账号，一般第三方库都会上传至PyPI,所以要先注册PyPI账号 <a href='https://pypi.org/'>注册界面</a>

2. 注册库，`python setup.py register` , 根据提示进行注册

3. 上传库，`twine upload dist/*`

#### 参考

> https://use-python.readthedocs.io/zh_CN/latest/packaging_and_sharing.html
