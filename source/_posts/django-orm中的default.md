---
title: Django Models中的default
date: 2019-08-11 20:32:02
tags:
    - Django
    - Python
categories:
    - 技术
---

在Django models中如果想给某个字段，设置默认值，可以使用default，如下

```python 
from django.db import models as db 

class User(db.Model): 

    name = db.CharField(default='', max_length=50, help_text='名称')
    age = db.IntegerField(help_text='年龄')
    ... 
```

#### 背景

不过Models中的默认值，只会作用在ORM层，就像官方文档所说，

> This can be a value or a callable object. If callable it will be called every time a new object is created. 

<!--more-->

```
user = User.objects.create(age=10).name  # ''
```

如果直接使用SQL创建时，并不会使用这里的默认值；

```sql
INSERT INTO USER (age) VALUES (20);
#  Field 'name' doesn't have a default value
```

这里试试手动加下SQL层的default，发现创建成功。

```sql
ALTER TABLE User ALTER COLUMN name SET DEFAULT '';
INSERT INTO USER (age) VALUES (30);
# success
```

> 当然实际项目中一般不会直接裸写SQL。

#### 潜在问题

不过通常正常发版时，一般会有以下几个步骤

- mi表
- 洗数据
- 灰度测试
- 发版

如果此次发版新增字段，只是设置了default，并未设置`null=True`，那么在mi完表，发版前这段时间，变动表的创建请求中的并不会包括新字段，所以创建数据时会报错，导致服务出错。

#### 解决办法

1. `migrate` 实际就是根据Migrations文件内容做操作，可以修改`makemigrations`生成的mi文件，使用Migration Operation，增加此字段SQL层的默认值。

2.  新增字段增加null=True，在之后的版本中去掉。
