---
title: 自动格式化工具 Black
date: 2019-06-02 21:33:54
tags:
    - Python
    - formatter
categories:
    - 工具
---

> Readability counts.               
>
> — The Zen of Python

最近了解到Black之后，项目的代码格式化工具，就从[Googel/yapf](https://github.com/google/yapf)换成了[Python/Black](https://github.com/python/black) ，~~大概喜新厌旧吧~~。

原项目中，用了`flake8+pylint+yapf`的三重检查，在替换过程中也遇到了一些问题，结果就是被pylint锤，被flake8锤，不过最终这些问题都在[github issue](https://github.com/python/black/issues)中找了解答。
> Ps: 官方解决方法：有冲突就ignore，disable，大不了remove

<!-- more -->

![](https://picture.wzmmmmj.com/star-trending.png)


## trailing comma

`Black`自动格式化后，发现换行后只有一行的，末尾并不会加","

```python
# in:

ImportantClass.important_method(exc, limit, lookup_lines, capture_locals, extra_argument)

# out:

ImportantClass.important_method(
    exc, limit, lookup_lines, capture_locals, extra_argument  # 这里并不会加 ","
)
```

结果就是，被`flake8-comma`爆锤`C812 missing trailing comma`，不过[解决办法](https://github.com/python/black/issues/551)也很简单，就是移除`flake8-comma`![](https://picture.wzmmmmj.com/commas.png)

## indentation

使用`yapf`期间，函数参数的排序一直是个难题，最近投票才决定使用了如下方式，`Black`则直接规定了一种方式

```python
# in:

def very_important_function(template: str, *variables, file: os.PathLike, engine: str, header: bool = True, debug: bool = False):
    """Applies `variables` to the `template` and writes to `file`."""
    with open(file, 'w') as f:
        ...
```

```python
# yapf
    
def very_important_function(template: str, 
                            *variables, 
                            file: os.PathLike, 
                            engine: str, 
                            header: bool = True, 
                            debug: bool = False):
    """Applies `variables` to the `template` and writes to `file`."""
    with open(file, 'w') as f:
        ...
```

```python
# black:

def very_important_function(
    template: str,
    *variables,
    file: os.PathLike,
    engine: str,
    header: bool = True,
    debug: bool = False,
):
    """Applies `variables` to the `template` and writes to `file`."""
    with open(file, "w") as f:
        ...
```

不过前提是需要全局`disable bad-continuation` ( [issue](https://github.com/python/black/issues/48)) 

![](https://picture.wzmmmmj.com/continuation.png)

## line breaks &&  slices

因为Black使用了更新的PEP8特性，导致的flake8会有一些奇怪的报错

- `W503 line break before binary operator` 
- `E203 whitespace before ':'`

解决办法依旧是[ignore](https://github.com/python/black#line-breaks--binary-operators)。

![](https://picture.wzmmmmj.com/slice.png)

## 总结

- yapf中，经常会有两次操作，两次不同的格式，导致统一格式需要另外配置与讨论，而Black则省去了这些步骤，强制性的统一格式化。

- Black中的可配置项少，yapf在这方面则具有很高的自由度。

- 虽然Black在格式方面强制性，不过字符长度是个例外，在一些特殊场景下，默许代码超出字符长度，如果想要改变格式化结果，可能需要在代码逻辑上进行改动。

