---
title: 记一次优化Vim启动速度
date: 2020-03-28 23:55:57
tags:
  - vim
---

最近宅家捣鼓了一段时间Vim，原打算试试Vim作为开发IDE，随着Vim Plugin增多，基本有个IDE的雏形，开发体验上有所改善，但也导致一个问题，启动Vim时变慢了，所以又花了点时间捣鼓优化。（PS：用什么Vim当IDE，Pycharm不香吗?）

## startuptime

在Vim7.2版本后自带了startuptime参数，可以输出Vim启动过程中的耗时

```shell
--startuptime {fname}                                   --startuptime
      During startup write timing messages to the file {fname}.
      This can be used to find out where time is spent while loading
      your .vimrc, plugins and opening the first file.
      When {fname} already exists new messages are appended.
      {only available when compiled with the +startuptime
      feature}
```
例如：

```shell
vim --startuptime time.log
vim --startuptime time.log ~/.vimrc
```

这里先上一组优化前的启动耗时
- 空白文件

```
221.131  000.229  000.229: sourcing /Users/a1/.vim/plugged/vim-airline/autoload/airline/extensions/tabline/builder.vim
244.400  025.347: first screen update
244.402  000.002: --- VIM STARTED ---
```

- 某千行python文件

```
375.848  000.497  000.497: sourcing /Users/a1/.vim/plugged/vim-gitgutter/autoload/gitgutter/hunk.vim
378.849  085.163: first screen update
378.863  000.014: --- VIM STARTED ---
```

此方法能粗略看出哪些操作启动耗时长，只能说够用。以下介绍两种更直观的启动耗时插件，便于优化。

###  startuptime.vim

```shell
Plug 'tweekmonster/startuptime.vim'
```

`startuptime.vim`会对当前文件多次启动，标记启动最慢的一些插件以及输出每个步骤的耗时，安装插件后，通过使用`:StartupTime`，输出startup-log，如图：

![startuptime.vim](https://picture.wzmmmmj.com/startuptime.vim.png)

### vim-startuptime

```shell
Plug 'dstein64/vim-startuptime'
```

`vim-startuptime`会返回启动时各个操作之间耗时以及百分比，同样使用`:StartupTime`，输出如图：

![vim-startuptime](https://picture.wzmmmmj.com/vim-startuptime.png)

### 发现问题

不难看出，拖慢startuptime的几处地方

1. 一些插件，例如nerdtree，vim-airline

2. filetype不对的插件没必要加载，例如打开python文件，显然没必要加载vim-go

3. 一些多余的autocommand

## 优化方案

### 懒加载

本人配置在vim启动后，nerdtree是处于未展开状态，显然启动时没必要加载，在需要使用加载即可，类似的，当filetype不同时，有些插件也没必要加载。

使用vim-plug的同学，一定熟悉一些plug option，像branch、do，其中有两种参数可实现lazy load。

```
on	On-demand loading: Commands or <Plug>-mappings
for	On-demand loading: File types
```

> 另一种插件管理 dein.vim 也可以实现lazy load

下面改写vimrc，对于命令触发的插件添加on参数，而对于fileType触发的则添加for参数。

```
Plug 'scrooloose/nerdtree', { 'on': 'NERDTreeToggle' }
Plug 'fatih/vim-go', { 'do': ':GoUpdateBinaries', 'for': 'go' }
```

改动后，startuptime.vim显示如下:

```
Total Time:   62.441 -- Flawless Victory

Slowest 10 plugins (out of 25)~
  vim-airline	12.314
    [runtime]	12.211
    [unknown]	10.636
     coc.nvim	6.997
     ...
```

执行`vim --startuptime time.log a.py`

```vim
323.386  051.707: first screen update
323.399  000.013: --- VIM STARTED ---
```

> nerdtree通过系统调用获取文件树，类似的还有statusline中获取分支名，这类系统调用的方式都是拖慢启动速度

### 替换插件

众所周知，vim-airline是statusline界的集大成者，兼容了许多常用插件，但同时也导致加载vim-airline时，加载不必要的内容，虽然[vim-airline performance](https://github.com/vim-airline/vim-airline#performance) 讲解了如何优化，不过我选择用更轻量的lightline.vim替换vim-airline。

```shell
" Plug 'vim-airline/vim-airline'
" Plug 'vim-airline/vim-airline-themes'
Plug 'itchyny/lightline.vim'
Plug 'mengelbrecht/lightline-bufferline'
```

如图为使用lightline.vim的statusline，以及配置对比

![lightline.vim](https://picture.wzmmmmj.com/lightline.vim.png)

再次使用`startuptime.vim` 以及`vim --startuptime time.log a.py`，输出如下

```
Total Time:   55.207 -- Flawless Victory

Slowest 10 plugins (out of 26)~
    [runtime]	12.493
    [unknown]	11.405
     coc.nvim	7.082
      darcula	4.432
          ale	3.546
vim-gitgutter	3.060
lightline.vim	2.748
     ...
```

```
204.982  025.837: first screen update
204.984  000.002: --- VIM STARTED ---
```

启动新文件

```
121.816  015.576: first screen update
121.818  000.002: --- VIM STARTED ---
```

## Over

至此，经过几个步骤之后，启动时间减少了越一半，200ms也还过得去，毕竟不是每次都打开千行的文件。另外对于现如今还不支持异步或者更新不积极的插件，如果有合适的替代品也可以替换，例如fzf.vim或Leaderf替换CtrlP，vista替换tagbar。定期更新vimrc，删除不需要的配置以及插件，有可能会有意外的惊喜哦。
