---
title: 浅谈Flask-sqlalchemy
date: 2018-10-28 20:08:58
tags:
    - Python
    - Flask
categories: 
    - 技术
---

> 最近看了点Flask，不同于Django的一小点，Flask没有自带ORM(其实大部分功能都没有)，不过如果需要操作数据库，可以使用第三方库flask-sqlalchemy ，大概用法与Django ORM 如出一辙。



## 配置sqlalchemy

首先当然是安装依赖啦`pip install flask-sqlalchemy`, 然后就是配置数据库

```python
import os

from flask import Flask
from flask_sqlalchemy import SQLAlchemy

basedir = os.path.abspath(os.path.dirname(__file__))
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///'+os.path.join(base_dir,'database.sqlite')

db = SQLAlchemy(app)
```

如果使用其他数据库其实就是改下`SQLALCHEMY_DATABASE_URI`, 以MySQL 为例：

```python
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql://f{username}:{password}@{hostname}/{database}'
```
<!-- more -->

## 定义模型

```python
from src import db


class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)
    role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))

    def __str__(self):
        return self.username


class Role(db.Model):
    __tablename__ = 'roles'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(60), unique=True)
    users = db.relationship('User', backref='role')

    def __str__(self):
        return self.name
```
> 如果想使用继承id图方便,外键部分需要使用primaryjoin<a href='https://stackoverflow.com/questions/52414291/accessing-a-mixin-member-variable-column-from-the-child'>一个坑</a>


其中 `__tablename__`可以重新定义表在库中的名字，`db.Column()` 为一个字段，第一个参数为字段类型如`db.String(32)`为字符串类型， `primary_key=True `字段为主键,(这里与Django orm 中会默认生成一个id作为主键不同), 其他类型 `unique=True`为唯一键，`index=True`为字段建立索引，`default=0`设置一个默认值，`nullable=True `该字段可为null 


## 数据库关系

数据库关系一般分为3种， 一对多，一对一， 多对多

#### 一对多

如果需要构建一对多关系，需要添加外键

```python
class User(BaseModel):
    #...
    role_id = db.Column(db.Integer, db.ForeignKey('role.id'))

class Role(BaseModel):
    #...
    users = db.realationship('User', backref='role')
```

这里给User表加了一个关联Role表的外键， 其中backref给role表做了一个反向引用

#### 一对一

如果是想将两表构成一对一只需要一步,realationship 中将`uselist=False`

```python
users = db.realationship('User', uselist=False, backref='role')
```

#### 多对多

多对多关系需要使用第三张表来建立关系，此处用官方文档的例子占个坑

```python
tags = db.Table('tags',
    db.Column('tag_id', db.Integer, db.ForeignKey('tag.id'), primary_key=True),
    db.Column('page_id', db.Integer, db.ForeignKey('page.id'), primary_key=True)
)

class Page(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    tags = db.relationship('Tag', secondary=tags, lazy='subquery',
        backref=db.backref('pages', lazy=True))

class Tag(db.Model):
    id = db.Column(db.Integer, primary_key=True)
```



## 数据库迁移

新建表或新增字段之后，需要做的第一步就是迁移数据库，在Django中两条命令即可

```shell
python manage.py migrate
python manage.py makemigrations
```

然而在Flask 中也不自带这些命令，如果用最原始的方式创建表，需要shell 中执行

```base
from src import db
db.create_all()  # 创建所有表
db.drop_all()    # 有更新需要删表重新建
db.create_all()
```

> 这样操作最大的问题不是麻烦，而是需要更新把原来的数据都删光了，感觉蠢的一匹~

所以这里要使用另一个第三方库`Flask-Migrate`,大致的操作跟Django差不多，

1. 安装依赖

```base
pip install flask-migrate
```

2. 第一步需要去添加配置

```python
from flask_migrate import Migrate

# ...
migrate = Migrate(app)
```

3. 第一次需要初始化项目，创建迁移仓库，会生成一个存放迁移文件的`migrations`文件夹

```base
flask db init
```

4. 每当有数据库更新时，需要创建迁移脚本

```base
flask db migrate -m 'migrate message'
```

5. 将迁移应用到数据库中

```base
flask db upgrade
```

6. 如果最后一次迁移未达到预期效果，可以还原上一次改动

```bash
flask db downgrade
# 删了最后一次的迁移文件,重新改动数据库,再重复3，4动作
```



## 数据库操作

大部分的增删改查操作，跟 Django ORM都没啥大区别，无非就是一些语法, 最主要应该就是sqlalchemy 需要`db.session.add()`  和 `db.session.commit()`

```python
# 增
role = Role(name='admin')
user1 = User(username='ZmJ', role=role)
user2 = User(username='zmj'role=role)
db.session.add_all([user1, user2])
db.session.commit()

# 查
user1 = User.query.filter_by(username='ZmJ').first()
user2 = User.query.filter_by(username='zmj').first()

# 改
user1.username = 'Flask'
db.session.add(user1)
db.session.commit()

# 删
db.session.delete(user2)
db.session.commit()
```

> 具体内容可查看官方文档
>
>  <a href='https://flask-migrate.readthedocs.io/en/latest/'>flask-migrate官方文档</a>
>
> <a href='http://flask-sqlalchemy.pocoo.org/2.3/'>flask-sqlalchemy官方文档</a>
