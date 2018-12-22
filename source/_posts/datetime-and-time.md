---
layout: datetime
title: Python 时间类型转化
date: 2018-12-22 18:34:36
tags:
    - Python
categories:
    - 技术
---

> 本文主要介绍 Python `time` 和`datetime`两个最常用的内置时间库中，不同时间类型之间相互转化

## Time

`time`库中主要分为3种时间类型

- 时间戳（Timestamp）
- 本地时间（struct_time），所处的时区时间
- 标准时间（struct_time），UTC时间

> 没错，国区时间 (本地时间) 与标准时间相差8小时

```python 
import time

print(time.time())  # 时间戳  1545479527.169141

print(time.localtime())  # 本地时间
# time.struct_time(tm_year=2018, tm_mon=12, tm_mday=22, tm_hour=19, tm_min=52, tm_sec=19, tm_wday=5, tm_yday=356, tm_isdst=0)

print(time.gmtime())  # utc时间
# time.struct_time(tm_year=2018, tm_mon=12, tm_mday=22, tm_hour=11, tm_min=52, tm_sec=26, tm_wday=5, tm_yday=356, tm_isdst=0)

print(time.ctime())  # 格式化时间
# 'Sat Dec 22 20:08:05 2018'
```

<!-- more -->

看下`time`中如何类型转化

```python
import time

timestamp = time.time()
struct_time = time.localtime()  # 或者time.gmtime()

# timestamp -> struct_time
print(time.localtime(timestamp))
print(time.gmtime(timestamp)) 

# struct_time -> timestamp
print(time.mktime(struct_time))

# timestamp -> str
print(time.ctime(timestamp)) 

# struct_time -> str
print(time.asctime(struct_time))
print(time.strftime("%Y-%m-%d, %H:%M:%S, %w", struct_time))  # 返回自定义格式

# str -> struct_time
str_time = time.strftime("%Y-%m-%d, %H:%M:%S, %w", struct_time)
struct_time = time.strptime(str_time, "%Y-%m-%d, %H:%M:%S, %w")  # 两个参数的指示意思要对应
```

>  参考下图食用更佳

![一张图看懂时间类型转化](http://picture.wzmmmmj.com/time.jpg)



## Datetime

`datetime`库主要包括4个类

- datetime.datetime
- datetime.date 
- datetime.time
- datetime.timedetla

> 每个类的具体差别请查看官方文档 <a href='https://docs.python.org/3/library/datetime.html'>Datetime</a>



### Datetime , struct_time 和 timestamp

```python
import datetime
import time


date_time = datetime.datetime.now()  # datetime 中的本地时间

# datetime -> timestamp
print(date_time.timestamp())

# datetime -> struct_time
print(date_time.timetuple())

# timestamp -> datetime
print(datetime.datetime.fromtimestamp(time.time()))

```

> 参考链接：
>
> https://zhuanlan.zhihu.com/p/23679915
>
> https://www.cnblogs.com/lxmhhy/p/6030730.html
>
> http://s9.sinaimg.cn/mw690/b09d4602xd10ea8f9ab88&690
