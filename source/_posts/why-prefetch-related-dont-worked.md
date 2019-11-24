---
title: 为什么Django prefetch_related不起作用
date: 2019-11-24 16:53:25
tags:
    - Django
    - Python
---

最近划水刷V站的时候，刷到[一个关于prefetch_related的问题](https://www.v2ex.com/t/618531)，简答了一下，虽然过程有点戏剧，不过让我重新看了一遍[官方文档](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#prefetch-related)，对prefetch_related也有了新的认识。对于刚接触prefetch_related，如果不了解其工作原理，相信都会遇到标题所示的问题。
**为什么我用了prefetch_related查询次数没减少，反而还多了？**

其实答案就在官方文档中

> Remember that, as always with QuerySets, any subsequent chained methods which imply a different database query will ignore previously cached results, and retrieve data using a fresh database query

<!--more-->

举个官方例子

```python
from django.db import models


class Topping(models.Model):
    name = models.CharField(max_length=30)

    
class Pizza(models.Model):
    name = models.CharField(max_length=50)
    toppings = models.ManyToManyField(Topping)
```
如果需要查询每个Pizza下的所有Toppings，如下

```python
pizzas = Pizza.objects.prefetch_related("toppings").all()
```
实际上这条语句除了执行 `SELECT "pizza"."id", "pizza"."name" FROM "pizza"`
因为加了prefetch_relate语句还会另外执行一条sql，通过in语句，将符合条件的pizza中所有toppings加载至内存中

```sql
SELECT
  ("pizza_toppings"."pizza_id") AS "_prefetch_related_val_pizza_id", 
  "topping"."id", "topping"."name" 
FROM "topping" 
INNER JOIN "pizza_toppings" 
ON ("topping"."id" = "pizza_toppings"."topping_id") 
WHERE "pizza_toppings"."pizza_id" IN (1, 2, 3)
```

所以通过执行类似`pizza.toppings.all()`的语句，实际上就会从缓存中读取，而不会产生sql，从而达到查询优化。但有时候总会有一些奇奇怪怪的需求。如果只想把topping.name='top'的读取出来，如果我们使用了这样的语句`pizza.toppings.filter(name='top')`，就会与上诉缓存不一致，从而再次执行一条sql，不但没有产生任何优化，反而因为用了prefetch_related，还多了一次查询。

> 除了filter()，order_by()，first() 也会因为查询不一致，用不上缓存，而count()则不会。如果不确定是否一致，可以输出语句执行sql瞅一瞅。如果使用类似`pizza.toppings.add()`等alter语句会使prefetch_related缓存被清除，从而导致查询优化失效，这种情况可以通过改变代码逻辑。

遇到这种情况，可以选择在内存中进行后续操作

```python
# filter
toppings = [
    topping
    for topping in pizza.toppings.all()
    if topping.name == 'top'
]

# first
topping = pizza.topping[0]

# order_by
toppings = sorted(pizza.toppings.all(), key=lambda _: _.id, reversed=True)
```
当然还可以通过自定义prefetch_related生成的sql，缓存需要的queryset

```python
from django.db.models import Prefetch

filters = Topping.objects.filter(name='top')
pizzas = Pizza.objects.prefetch_related(Prefetch("toppings", queryset=filters))                                     
```
如果想对同一个外键进行不用的filter，还可以使用参数to_attr命名

```python
filters = Topping.objects.filter(name='top')
order = Topping.objects.order_by('-id')
pizzas = Pizza.objects.prefetch_related(
    Prefetch("toppings", queryset=filters, to_attr='filter_toppings'),
    Prefetch("toppings", queryset=order, to_attr='order_toppings'),
)
```
这样通过pizzas.order_toppings.all()或pizzas.filter_toppings.all()使用即可

