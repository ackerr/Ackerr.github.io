---
title: Django 中的aggregate、annotate以及Q、F查询
date: 2019-01-12 20:19:06
tags:
   - Python
   - Django
categories:
   - 技术
---

日常工作使用中，大部分的条件查询通过`filter`都可以完成，但是在更复杂的查询时，`filter`就显得无力了。

不过，好在Django为我们提供了一些可以进行复杂查询的语句，比如`aggregate, annotate, Q, F`


## models

> 来自<a href='https://docs.djangoproject.com/en/2.1/topics/db/aggregation/'>Django官方文档</a>的`models`，**本文的查询都基于这些models**

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    age = models.IntegerField()

class Publisher(models.Model):
    name = models.CharField(max_length=300)

class Book(models.Model):
    name = models.CharField(max_length=300)
    pages = models.IntegerField()
    price = models.DecimalField(max_digits=10, decimal_places=2)  # 原价
    member_price = models.DecimalField(max_digits=10, decimal_places=2)  # 会员价
    rating = models.FloatField()
    authors = models.ManyToManyField(Author)
    publisher = models.ForeignKey(
        Publisher, on_delete=models.CASCADE)
    pubdate = models.DateField()

class Store(models.Model):
    name = models.CharField(max_length=300)
    books = models.ManyToManyField(Book)
```



## aggregate（聚合查询）

在原生SQL语句中常常会用到一些聚合函数，用来对一组数据进行计算，比如`AVG/COUNT/MAX/MIN/SUM` 等， `aggregate`则是`Django`中实现聚合函数的。以下用几个简单例子来说明。

**例：求某个出版社出售的最便宜的一本书**

```python
from django.db.models import MIN

Book.objects.filter(publisher__name='新华出版社').aggregate(min_price=Min('price'))
```

> aggregate 后的结果不再为QuerySet， 而是相应的{min_price=20.00}， 如果未设置字段名，会默认为`查询字段__聚合方法`，例如`price__min`

当然如何你想聚合其他值，比如平均价格或者最高评分，可以添加参数

```python
from django.db.models import Avg, Max, Min

Book.objects.filter(publisher__name='新华出版社').aggregate(min_price=Min('price'), avg_price=Avg('price'), max_rate=Max('rating'))
```
<!-- more -->

## annotate（分组查询）

`annotate`，顾名思义就是为`QuerySet`中的每个对象生成一个独立的统计值， 统计值会存在`QuerySet`。

**例：统计每个出版社出版的图书数量，并且以倒序排列**

```python
from django.db.models import Count

pubs = Publisher.objects.annotate(book_num=Count('book').order_by('-book_num')
```

> 通过 pubs[0].book_num 来查看结果

**例：查询每个出版社出的书的总价格**

```python
from django.db.models import Sum

pubs = Publisher.objects.annotate(
    total_price=Sum('book__price')).values('name', 'total_price')
```



## Q查询

`filter()` 等方法中的关键字参数查询 都是进行`AND`运算，如果需要执行类型`OR`语句，就需要使用`Q查询`了。

当多个条件查询时，还可以通过`&`， `|` 以及`()`进行分组编写Q对象，而且Q对象还可以通过使用 `~` 表达取反（NOT），当然如果查询较简单也可以使用exclude。

**例：查询价格是小于20或大于50的图书**

```python
from django.db.models import Q

Book.objects.filter(Q(price__lt=20) | Q(price__gt=50))
```

**例：查询出版社是新华出版社或者清华出版社并且不是2018年出版的书**

```python
from django.db.models import Q

Book.objects.filter(
(Q(publisher__name='新华出版社')|Q(publisher__name='清华出版社'))& ~Q(pubdate__year=2018))
```

如果想`Q查询`与关键字参数同时使用，需要遵循一个条件，Q对象必须位于所有关键字参数前面，如下所示

**例：查询新华出版社评分在5-9之间的图书**

```python
from django.db.models import Q

Book.objects.filter(Q(rating__gt=5)|Q(rating__lt=9), publisher__name='新华出版社')
```



## F查询

在上面的例子中，我们只是将字段值与某个常量作比较进行筛选，如果需要对字段值之间比较筛选，就需要使用Django 提供的`F查询`。

**例：查询会员价与原价相同的图书**

```python
from django.db.models import F

Book.objects.filter(price=F('memeber_price'))
```

当然`F语句`还可以更改一列值，例如将选中的所有图书原价增加10块

```python
Book.objects.all().update(price=F('price')+10)
```

与`annotate`查询一起使用，**求原价与会员价的差价**

```python
from django.db.models import F

Book.objects.all().annotate(spread_price=F('price') - F('memeber_price'))
```



## 总结

这4种查询已经可以应付大多数的场景，对不同的场景可以有不同的组合方式，在我看来：

- `aggregate`用于聚合统计数据；
- `annotate`用于对数据进行分组，类似`group by`；

- `Q()`是对`QuerySet`进行更复杂的过滤筛选；

- `F()`用于对`Query`字段间的某列值进行比较或操作。



> 本文只是简单的介绍了4种查询，更多内容可查阅<a href='https://docs.djangoproject.com/en/2.1/topics/db/aggregation/'>Django官方文档</a>
