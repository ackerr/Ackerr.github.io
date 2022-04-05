---
title: Git commit 配置 flake8
date: 2018-10-11 19:49:03
tags:
    - git
categories: 
    - 工具
---


> 我司后端使用flake8来检测代码规范，所以正常情况下都会在提PR前`flake8`一下 ，不过偶尔也会忘记，最终导致GitLab上的pipeline挂了。

由于我司主站的代码量有点大，每次`flake8 .`需要等待很长时间，如果最后还有代码规范问题，就很难受了。
但是其实我们每次更改的文件就那么几个，没必要把所有代码都检查一遍，只需要检查更改的几个文件。

<!-- more -->
## 初始版本

因为`git status ` 可以查看到每次更新的文件，所以在`git add `前，跑执行`flake8 xxx yyy`，但是文件多起来这样搞就显得很蠢，有没有什么方法可以获取所有更新文件呢？答案是`git diff --name-only HEAD~` ，会返回与上次不同的文件，所以每次就可以输入

```bash
flake8 `git diff --name-only HEAD~`
```

但每次都这么输会比较麻烦，所以我们可以给这条命令在`~/.bash_profile`(或者在你习惯加alias的文件)中加一条alias。

```bash
vim ~/.bash_profile

alias flake='flake8 `git diff --name-only HEAD~`'
```

这样就可以让自己每次检查更改的文件，不过这样做只能让自己更便利的检查，而不能避免忘记。

## 升级版本

**因为在`.git/hooks`中的pre-commit 文件中，可以配置在`git commit`执行前的操作**

为了避免忘记，我们可以在每次执行`git commit`时，自动先执行flake8，如果检测未通过，会报错，需要重新提交。

具体操作如下：

```bash
vim .git/hooks/pre-commit

# 如果提示不能更改文件，需要加一步操作即可
chmod u+x .git/hooks/pre-commit
```

写上自己想commit前操作的脚本（这里我们只需要两句话即可）

```bash
#!/bin/sh
flake8 `git diff --cached --name-only`
```



## 总结

简单的说就是三个步骤：

1. pip install flake8

1. chmod u+x .git/hooks/pre-commit

1. vim .git/hooks/pre-commit



## 效果演示

![效果演示](/images/flake8.png)

