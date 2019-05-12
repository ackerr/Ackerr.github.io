---
title: Django ORM的默认JOIN方式
date: 2019-05-12 23:16:23
tags:
    - Django
    - Python
categories:
    - 技术
---

### SQL的JOIN操作

对于SQL的JOIN操作，无非就这么几种

- INNER JOIN
- OUTER JOIN 
- LEFT JOIN 
- RIGHT JOIN
- CROSS JOIN 

通过[文氏图](https://blog.codinghorror.com/a-visual-explanation-of-sql-joins/)，可以更清晰的了解每种操作以及所得的结果集。

<!--more-->

![](https://picture.wzmmmmj.com/sql.jpg)

### Django ORM的JOIN操作

> Django跨表查询时，使用`select_related()`，将相关表`JOIN`，从而减少查库次数。

**那Django会使用哪种`SQL JOIN`呢？**

Django 定义两张表

```python
from django.db import models


class Teacher(models.Model):
    name = models.CharField(max_length=50)


class Student(models.Model):
    name = models.CharField(max_length=50)
    teacher = models.ForeignKey(
        Teacher, related_name='teacher', on_delete=models.DO_NOTHING
    )
```

初始化后，`python manage.py shell`中执行, 输出相关SQL

```python
from demo.models import *

students = Student.objects.select_related('teacher')
print(students.query)

SELECT `student`.`id`, `student`.`name`, `student`.`teacher_id`, `teacher`.`id`, `teacher`.`name` 
FROM `student` 
INNER JOIN `teacher` ON (`student`.`teacher_id` = `teacher`.`id`)
```

将表`Student.teacher`字段`null=True`，重复上一步操作，输出SQL如下

```python
SELECT `student`.`id`, `student`.`name`, `student`.`teacher_id`, `teacher`.`id`, `teacher`.`name` 
FROM `student` 
LEFT OUTER JOIN `teacher` ON (`student`.`teacher_id` = `teacher`.`id`)
```

由此可知，在ForeignKey `null=True`时，关联查询会通过`LEFT JOIN`，将左表数据查出，没有匹配的数据则用null代替。反之则通过`INNER JOIN`筛选出两表并集。
