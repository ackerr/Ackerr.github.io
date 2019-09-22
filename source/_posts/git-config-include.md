---
title: 不同目录配置不同git config
date: 2019-09-22 16:44:52
tags:
    - git
categories:
    - 工具
---

日常使用git时，一般会有全局配置文件的`.gitconfig`，所有repo会默认使用这个配置。如果需要特殊配置某个repo，只需要修改repo里的`.git/config`文件即可。但如果需要修改repo数量变多就容易忘，导致提交错误author的commit。

### Demo

例如，公司repo与个人repo，想使用不同的`user.name`与`user.email`，简单操作就是，修改个人repo中`.git/config`中的配置

```shell
[user]
    name = name1
    email = name1@example.com
```

或执行以下语句，修改配置（两种方法本质上是一样的）

```bash
git config user.name name1
git config user.email name1@example.com
```

这样就会覆盖全局配置，但repo一多，这种方法就显得效率很低。

<!--more-->

### Solution

个人会用不同的文件夹区分公司与个人的repo，这样就可以通过[git conditional include](https://git-scm.com/docs/git-config#_conditional_includes)，给不同目录下的repo，设置不同的`git config`。假设公司repo，会放在`work`目录下。

新建一个`.gitconfig-work`，内容如下

```shell
[user]
    name = name2
    email = name2@example.com
```

打开全局配置`.gitconfig`文件，添加目录配置

```shell
[includeIf "gitdir:~/work/"]
    path = .gitconfig-work
```

这样`work`下的新repo，都会使用`.gitconfig-work`中的配置覆盖全局配置，从而达到不同目录使用不同的`git config`

### Tip

对于work目录下已存在的repo，可执行以下命令查看`.gitconfig-work`中配置是否生效

```bash
git config --shwo-origin --get user.name
git config --shwo-origin --get xxxx
```

> 参考
>
> https://git-scm.com/docs/git-config#_conditional_includes
>
> https://stackoverflow.com/questions/43919191/git-2-13-conditional-config-on-windows
