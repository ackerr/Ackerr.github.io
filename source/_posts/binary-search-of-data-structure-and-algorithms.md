---
title: 数据结构与算法之二分查找
date: 2019-03-30 14:26:32
tags:
    - 查找
categories:
    - 数据结构与算法
---

### 关于二分查找

> 二分查找 (binary search) ，是在**有序数组**中查找某一特定元素的搜索算法。

**有序数组**是二分查找的前提条件，如果数据结构是由链表构成，使用二分查找只会事倍功半。

其原理是将查找元素跟区间中间的元素对比，将待查找的区间缩小为之前的一半，直到找到要查找的元素，或者区间被缩小为0。

假设数据集为n，每次查找后数据减少一半，如图，其中可知k=log<sub>2</sub>n，

![](/images/bsearch.jpg)

所以二分查找的时间复杂度为 **O(logn)**， 并且不需要额外空间，空间复杂度为 **O(1)**

<!-- more -->
### 二分查找基本实现

> 有序数组不存在重复元素

#### 非递归

```python
def binary_search(nums, n):
    low, high = 0, len(nums) - 1
    while low <= high:
        mid = (low + high) // 2
        if nums[mid] < n:
            low = mid + 1
        elif nums[mid] > n:
            high = mid - 1
        else:
            return mid
    return -1
```

#### 递归

```python
def binary_search(nums, n):
    return _search(nums, 0, len(nums) - 1, n)

def _search(nums, low, high, n):
    if low > high:
        return -1
    mid = (low + high) // 2
    if nums[mid] < n:
        return _search(nums, mid+1, high, n)
    elif nums[mid] > n:
        return _search(nums, low, mid-1, n)
    else:
        return mid
```

### 二分查找变种的实现

基本实现中的事例有个前提条件**需要数组不存在重复元素**，当给定的数组中包含重复的元素时，并且有查找元素条件时，使用上面事例代码查找，显然不能达到要求，不过稍加修改就可以使用，以下是4道变形题。

#### 查找第一个值等于给定值的元素

```python
def bs_first(nums, n):
    low, high = 0, len(nums) - 1
    while low <= high:
        mid = (low + high) // 2
        if nums[mid] > n:
            high = mid - 1
        elif nums[mid] < n:
            low = mid + 1
        else:  # nums[mid] == n
            if mid == 0 or nums[mid - 1] < n:
                return mid
            else:
                high = mid - 1
    return -1
```

#### 查找最后一个值等于给定值的元素

```python
def bs_last(nums, n):
    low, high = 0, len(nums) - 1
    while low <= high:
        mid = (low + high) // 2
        if nums[mid] > n:
            high = mid - 1
        elif nums[mid] < n:
            low = mid + 1
        else:  # nums[mid] == n
            if mid == len(nums) - 1 or nums[mid + 1] > n:
                return mid
            else:
                low = mid + 1
    return -1
```

以上两道变形题，查找重复元素中的第一个值（最后一个值），如代码所示，只需要在`nums[mid] == n`分支，多一个判断，查看前一位(后一位)是不是指定值，如果是则继续查找。

------

#### 查找第一个大于或等于给定值的元素

> 优先查找第一个等于给定值元素，如果不存在，则查找第一个大于该值的元素

```python
def bs_first_equal_or_gather(nums, n):
    low, high = 0, len(nums) - 1
    while low <= high:
        mid = (low + high) // 2
        if nums[mid] >= n:
            if mid == 0 or nums[mid - 1] < n:
                return mid
            else:
                high = mid - 1
        elif nums[mid] < n:
            low = mid + 1
    return -1
```

#### 查找最后一个小于或等于给定值的元素

> 优先查找最后一个等于给定值元素，如果不存在，则查找最后一个小于该值的元素

```python
def bs_last_equal_or_less(nums, n):
    low, high = 0, len(nums) - 1
    while low <= high:
        mid = (low + high) // 2
        if nums[mid] > n:
            high = mid - 1
        elif nums[mid] <= n:
            if mid == len(nums) - 1 or nums[mid + 1] > n:
                return mid
            else:
                low = mid + 1
    return -1
```

以上两道变形题，只需要注意该值 是不是 **大于等于 (小于等于)** 给定值，如果是，再判断该值的**上一个值小于 (下一个值大于)**给定值， 如果是则返回该值。



### 总结

二分查找相对于其他算法简单，不过要想写出对的二分查找，需要注意一下几点。

1. 循环的退出条件，`low <= high`；

2. 每种条件的判断，以及low，high的更新，一般的变形题只需要更改判断条件及区间的更新即可；

3. 中间值mid的取值，如果在low，high 值较大时，可使用位运算进行优化 `mid = low + ((high - low) >> 1)`。

