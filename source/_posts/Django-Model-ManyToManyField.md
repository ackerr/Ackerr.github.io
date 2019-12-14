---
title: Django Model中的ManyToManyField
date: 2019-12-14 21:35:11
tags:
    - Django
    - Python
---

日常开发中，设计表结构经常会遇到多对多关系，都会立刻想到`ManyToManyField`，例如用户收藏门店，用户可以收藏多个门店，门店也可以被多个用户收藏，这个场景就很适合使用`ManyToManyField`。Model如下：

```python
from django.db import models

class Account(models.Model):
    name = models.CharField(max_length=20, help_text="用户名")
   
class Shop(models.Model):
    name = models.CharField(max_length=20, help_text="门店名")
    collectors = models.ManyToManyField(Account, help_text="收藏用户")
```
<!--more-->

依次完成makemigration跟migrate后，创建几条记录

```python
kfc = Shop.objects.create(name="kfc")
alice = Account.objects.create(name="alice")
kfc.collectors.add(alice)
kfc.collectors.create(name="bob")
```

通过`bob.shops.all()`或`kfc.accounts.all()`即可获取bob收藏的门店和收藏kfc的所有用户。但如果添加数据时，使用以下步骤，则会抛错ValueError

```python
carol = Account(name="carol")
kfc.collectors.add(carol)
# ValueError: Cannot add "<Account: Account object (None)>"
# : instance is on database "default", value is on database "None"

burger_king = Shops(name="burger king")
burger_king.collectos.add(alice)  # ValueError
```

此时查看数据库tables，不难发现，Django通过ManyToManyField生成了一张中间表名为`*_shop_collectors`的中间表。上面操作报错也就不难理解，因为当前carol和burger_king还未存在db中。

### “through” model

如果需求变更，还需要记录用户收藏门店的时间，只使用默认的ManyToManyField显然是不够的。ManyToManyField提供了“through”字段，支持自定义中间表

1. 将默认中间表写成Django Model

```python
class Shop(models.Model):
    name = models.CharField(max_length=20, help_text="门店名")
    collectors = models.ManyToManyField(Account, through="ShopCollector")
    
class ShopCollector(models.Model):
    class Meta:
        db_table = "<table_name>"
    account = models.ForeignKey(Account, on_delete=models.CASCADE)
    shop = models.ForeignKey(Shop, on_delete=models.CASCADE)
```

> 中间表未生成，可以跳过第二步，如果需要在原有中间表上修改，db_table需要是原表名

2. 执行`makemigration`，修改生成的mi文件

```python
state_operations = [...]  # 原operations改为state_operations
operations = [
    migrations.SeparateDatabaseAndState(state_operations=state_operations)
]
```

3. 加上collect_at字段，依次执行makemigrations，migrate

```python
class ShopCollector(models.Model):
    class Meta:
        db_table = "<table_name>"
    account = models.ForeignKey(Account, on_delete=models.CASCADE)
    shop = models.ForeignKey(Shop, on_delete=models.CASCADE)
    collect_at = models.DateTimeField(auto_now_add=True)
```

4. 依次执行makemigrations，migrate

几步操作完后，新建collector时，都会附带收藏时间

### through_field

ManyToManyField生成的中间表只会有两个外键，从而Django可知哪两个model存在多对多关系，但使用自定义中间表时，新增了外键，如下存在两个除Shop表的两个外键：

```python
class ShopCollector(models.Model):
    shop = models.ForeignKey(Shop, on_delete=models.CASADE)
    account = models.ForeignKey(Account, on_delete=models.CASCADE)
    inviter = models.ForeignKey(
        Account, 
        related_name="invite_collectors",
        on_delete=models.CASADE,
    )
    collect_at = models.DateTimeField(auto_now_add=True)
```
这种情况需要通过设置`through_field`为两个参数组成的元祖(field1, field2)，并且第一个元素需要是定义ManyToManyField表的外键字段名，声明多对多关系的两个外键，这个例子需要将collectors改为

```python
collectors = models.ManyToManyField(
    Account, 
    through="ShopCollector", 
    through_fields=("shop", "account"),
)
```
> 对于自关联的自定义中间表，最好也通过through_fields显示声明

### symmetrical

当使用自关联的多对多关系时，如下

```python
class Account(models.Model):
    name = models.CharField(max_length=20, help_text="用户名")
    followers = ManyToManyField("self", related_name="followees")
```
此时所有Account的followers都是对称效果，也就是bob是alice的朋友，那alice也是bob的朋友，而且此时reverse relationship无效，不过可以通过设置`symmetrical=False`取消，使用reverse relatationship，例如这里的followers和followees。
