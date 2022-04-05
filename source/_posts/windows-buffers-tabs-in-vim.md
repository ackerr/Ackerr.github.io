---
title: Vim中的Buffers, Windows, Tabs
date: 2020-01-31 19:50:06
tags:
    - vim
---

Vim中的buffer，tab和window是容易混淆的概念，vim中输入`:help window`，会对三个概念进行简短解释:

> Summary:
> A buffer is the in-memory text of a file.
> A window is a viewport on a buffer.
> A tab page is a collection of windows.

对于常用IDE的同学，如果把这几个概念带入IDE中，常会摸不着头脑 ~~比如我~~。对照下图，应该更容易理解
![Tabs Windows Buffers](/images/vim-tabs-windows-buffers.jpg)

<!--more-->

### Buffers
> A buffer is the in-memory text of a file.

Buffer即加载在内存中内容，常为一个文件，并且一个文件只会存在一个buffer。通过`vim a.txt b.txt`会打开两个文件buffer，同时会在编辑a.txt，接着执行`:edit c.txt`，窗口会显示c.txt文件，通过 `:ls` 可以显示所有buffer，这里会显示三行记录。

```vim
:ls
  1 #h   "a.txt"                        line 1
  2      "b.txt"                        line 0
  3 %a   "c.txt"                        line 1
```

可以看到a.txt，c.txt前面会有标识，相关标识含义如下：

```
– （非活动的缓冲区）
a （当前被激活缓冲区）
h （隐藏的缓冲区）
% （当前的缓冲区）
# （交换缓冲区）
= （只读缓冲区）
+ （已经更改的缓冲区）
```

所以%a表示当前激活的buffer，而#h则表示上一个展示的buffer且是隐藏的。
通过`:b <编号>`或`:b <文件全称>`可切换到相应的buffer，当然也可以通过`:bprev`或`:bnext`前后切换buffer，`:bdelete <编号或文件全称>`删除某个buffer。如果vimrc中声明`set hidden`，则在文件修改后未保存可以切换至其他buffer，而如果显式声明`set nohidden`则需要先保存。

#### Tips
如果使用NERDTree时出现:bd退出vim session的时候，可以看看[此issue](https://github.com/preservim/nerdtree/issues/400)，通过`:bp<CR>bd #<CR>`可以达到删除当前buffer，并回到上一个buffer的效果。

### Windows
> A window is a viewport on a buffer.

Window即用于展示buffer的窗口，通过该窗口才能看到buffer中的内容。 使用`:split`或`:vsplit`，可实现分屏效果，在新窗口中展示当前buffer，也可在操作后加文件，新窗口中展示新打开的buffer。也就是说，多个window可以展示相同的buffer。

### Tabs

> A tab page is a collection of windows.

Tab即多个window的组合，一个tab中可以展示多个window，通过`:tabp/:tabn 或 gT/gt`前后切换tab，以及`:tabc`关闭当前tab，更多操作可见`:help tab-page`。

使用tab的推荐做法是进行分组管理，例如无关项目或不同类型的代码(server端与client端)都可以在多个tab中管理。在buffer中提到一个文件只会存在一个buffer，所以在多个tab的相同buffer，其实他们是同一个buffer，在某一处被关闭时，其他tab中也相应的不可见。

当然，不同人对三者的看法与使用习惯是不同的，例如有些人喜欢一个tab对应一个buffer，类似IDE中的布局，也有人不使用tab，对于什么场景下使用buffer还是tab，因人而异，一句话就是怎么舒服怎么来。

> 参考：
> [buffers or tabs](https://stackoverflow.com/questions/26708822/why-do-vim-experts-prefer-buffers-over-tabs)
> [using vim tabs like buffers](https://stackoverflow.com/questions/102384/using-vims-tabs-like-buffers/103590#103590)
> [buffers windows tabs](https://sanctum.geek.nz/arabesque/buffers-windows-tabs/)
> [vim tab madness buffers vs tabs](https://joshldavis.com/2014/04/05/vim-tab-madness-buffers-vs-tabs/)
