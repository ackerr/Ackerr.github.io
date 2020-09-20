---
title: Golang实现git-open 
date: 2020-09-20 23:44:28
tags:
   - go
   - cli
   - git
---

[git-open](https://github.com/paulirish/git-open)是GitHub上开源的一个git相关命令行工具，终端通过键入`git open`，会在浏览器中打开当前路径所在git项目的远端页面，支持多种代码托管平台，具体可查看官方文档。本质上git-open是一个shell脚本，核心代码仅有200+行。在无参数的前提下，主要有以下几步

1. 判断当前路径是否在git工作区内
2. 获取当前branch的remote-url
3. 转换remote-url为托管仓库的页面地址
4. 根据不同平台，命令行唤起浏览器

下文将依照以上步骤，通过Golang第三方库cobra来实现类似命令行功能，具体cobra用法，可移步[官方文档](https://github.com/spf13/cobra)

> 完整代码放置文章结尾，git open功能也已集成至[ackerr/lab](https://github.com/ackerr/lab)。 

## Step.0

首先建立好基础代码结构，并简单实现了两个工具函数
- GitCommand: 简单封装Golang中执行git命令行
- Err: 返回error时，输出报错信息，直接退出程序

具体代码如下

```go
package main

import (
	"github.com/spf13/cobra"
	"os"
	"os/exec"
)

func main() {
	if err := rootCmd.Execute(); err != nil {
		os.Exit(1)
	}
}

// 定义命令
var rootCmd = &cobra.Command{
	Use:  "git-open",
	Run: openCurrentRepo,
}

// 核心逻辑
func openCurrentRepo(cmd *cobra.Command, args []string) {
    // do something
}

// 稍微封装一下git command
func GitCommand(args ...string) (string, error) {
	cmd := exec.Command("git", args...)
	output, err := cmd.Output()
	return string(output), err
}

// 快捷退出
func Err(msg ...interface{}) {
	fmt.Println(msg...)
	os.Exit(1)
}
```

##  Step.1

通过`git rev-parse`，获取当前路径与.git目录相关信息，例如以下命令

**判断当前路径是否在git工作区内**

```shell
git rev-parse --is-inside-work-tree
```

**获取.git目录路径**

```shell
git rev-parse --git-dir
```

**获取git项目绝对路径**

```shell
git rev-parse --show-toplevel
```

> 更多git rev-parse命令可查看[官方文档](https://git-scm.com/docs/git-rev-parse)

从以上命令挑选一个判断当前是否在工作区中即可，伪代码如下

```go
func CurrentGitRepo() (string, error) {
	output, err := GitCommand("rev-parse", "-q", "--show-toplevel")
	return output, err
}
```

## Step.2

在git中，通过HEAD指针来指向当前分支或指向某个commit，来确定当前索引位置，记录在.git/HEAD文件中，而通过`git symbolic-ref`命令，可获取当前HEAD信息

**获取当前分支名**

```shell
git symbolic-ref --short HEAD
```

**获取当前分支remote**

```shell
git config branch.<branch>.remote
```

**获取remote-url**

```shell
git ls-remote --get-url <remote>
```

返回的remote-url即执行`git remote add <remote-url>`时，添加的那串url，一般有https或ssh两种形式，格式如下

```
# ssh  
git@github.com:Ackerr/lab.git

# https
https://github.com/Ackerr/lab.git
```

伪代码如下

```go
// 获取当前分支，如果不存在则使用master
func CurrentBranch() string {
	branch, err := SymbolicRef("HEAD", true)
	if branch == "" || err != nil {
		branch = "master"
	}
	return branch
}

// 执行symbolic-ref命令
func SymbolicRef(ref string, short bool) (string, error) {
	args := []string{"symbolic-ref"}
	if short {
		args = append(args, "--short")
	}
	args = append(args, ref)
	output, err := GitCommand(args...)
	return firstLine(output), err
}

// 通过分支获取当前remote，默认为origin
func CurrentRemote(branch string) string {
	remote, err := GitCommand("config", fmt.Sprintf("branch.%s.remote", branch))
	remote = firstLine(remote)
	if remote == "" || err != nil {
		remote = "origin"
	}
	return remote
}

// 通过remote获取remote-url，如果使用错误的remote，则报错退出
func RemoteURL(remote string) string {
	gitURL, err := GitCommand("ls-remote", "--get-url", remote)
	if err != nil {
		Err("git remote is not set for", remote)
	}
	gitURL = firstLine(gitURL)
	if gitURL == remote {
		Err(remote, "is a wrong remote")
	}
	return gitURL
}

// 移除末尾换行符
func firstLine(output string) string {
	if i := strings.Index(output, "\n"); i >= 0 {
		return output[0:i]
	}
	return output
}
```

> 需要注意的一点，通过exec执行的git命令，返回值会在末尾添加换行符'\n'，所以每个命令结束都需要调用firstLine移除

## Step.3

正常情况下，远端仓库页面地址结构为 `<protocol>/<domain>/<repo path>`，一般情况下protocol为https，GitLab与GitHub类型，转换代码如下

```go
// for example:
// git@github.com/Ackerr:lab.git     -> https://github.com/Ackerr/lab            
// https://github.com/Ackerr/lab.git -> https://github.com/Ackerr/lab
func TransferToURL(gitURL string) string {                                         
    var url string
    if strings.HasPrefix(gitURL, "https://") {
        url = gitURL[:len(gitURL)-4]
    }
    if strings.HasPrefix(gitURL, "git@") {
        url = gitURL[:len(gitURL)-4]
        url = strings.Replace(url, ":", "/", 1)
        url = strings.Replace(url, "git@", "https://", 1)
    }
    return url
}
```

## Step.4

对于不同操作系统，可能使用不同的命令唤起浏览器，例如macOS默认通过open，而windows则通过start，当然也可以通过设置环境变量$BROWSER，切换默认命令，代码如下

```go
func searchBrowserLauncher() (browser string) {
    switch runtime.GOOS {
    case "darwin":
        browser = "open"
    case "windows":
        browser = "cmd /c start"
    case "linux":
        browser = "xdg-open"
    default:
        browser = ""
    }
    return browser
}
```

通过以上几步，git open的核心伪代码大致完成，接下来微调即可，完整代码如下

```go
package main

import (
	"errors"
	"fmt"
	"github.com/kballard/go-shellquote"
	"github.com/spf13/cobra"
	"os"
	"os/exec"
	"runtime"
	"strings"
)

func main() {
	if err := rootCmd.Execute(); err != nil {
		os.Exit(1)
	}
}

var rootCmd = &cobra.Command{
	Use:  "git-open",
	Run: openCurrentRepo,
}

func openCurrentRepo(cmd *cobra.Command, args []string) {
	var remote string
	if len(args) > 0 {
		remote = args[0]
	}
	_, err := CurrentGitRepo()
	if err != nil {
		Err("not a git repository")
	}
	if remote == "" {
		branch := CurrentBranch()
		remote = CurrentRemote(branch)
	}
	gitURL := RemoteURL(remote)
	url := TransferToURL(gitURL)
	_ = OpenBrowser(url)
}

func GitCommand(args ...string) (string, error) {
	cmd := exec.Command("git", args...)
	cmd.Stderr = os.Stderr
	output, err := cmd.Output()
	return string(output), err
}

// 获取当前git项目路径
func CurrentGitRepo() (string, error) {
	output, err := GitCommand("rev-parse", "-q", "--show-toplevel")
	return output, err
}

// 获取当前分支，如果不存在则使用master
func CurrentBranch() string {
	branch, err := SymbolicRef("HEAD", true)
	if branch == "" || err != nil {
		branch = "master"
	}
	return branch
}

func SymbolicRef(ref string, short bool) (string, error) {
	args := []string{"symbolic-ref"}
	if short {
		args = append(args, "--short")
	}
	args = append(args, ref)
	output, err := GitCommand(args...)
	return firstLine(output), err
}

// 通过分支获取当前remote，默认为origin
func CurrentRemote(branch string) string {
	remote, err := GitCommand("config", fmt.Sprintf("branch.%s.remote", branch))
	remote = firstLine(remote)
	if remote == "" || err != nil {
		remote = "origin"
	}
	return remote
}

// 通过remote获取remote-url，如果使用错误的remote，则报错退出
func RemoteURL(remote string) string {
	gitURL, err := GitCommand("ls-remote", "--get-url", remote)
	if err != nil {
		Err("git remote is not set for", remote)
	}
	gitURL = firstLine(gitURL)
	if gitURL == remote {
		Err(remote, "is a wrong remote")
	}
	return gitURL
}

// 移除末尾换行符
func firstLine(output string) string {
	if i := strings.Index(output, "\n"); i >= 0 {
		return output[0:i]
	}
	return output
}

// remote url转换为web url
func TransferToURL(gitURL string) string {
	var url string
	if strings.HasPrefix(gitURL, "https://") {
		url = gitURL[:len(gitURL)-4]
	}
	if strings.HasPrefix(gitURL, "git@") {
		url = gitURL[:len(gitURL)-4]
		url = strings.Replace(url, ":", "/", 1)
		url = strings.Replace(url, "git@", "https://", 1)
	}
	return url
}

// 输出报错，结束程序
func Err(msg ...interface{}) {
	fmt.Println(msg...)
	os.Exit(1)
}


// 唤起浏览器，打开url
func OpenBrowser(url string) error {
	launcher, err := browserLauncher()
	if err != nil {
		return err
	}
	args := append(launcher, url)
	cmd := exec.Command(args[0], args[1:]...)
	return cmd.Run()
}

// 切分命令为数组，方便exec.Command执行
func browserLauncher() ([]string, error) {
	browser := os.Getenv("BROWSER")
	if browser == "" {
		browser = searchBrowserLauncher()
	} else {
		browser = os.ExpandEnv(browser)
	}

	if browser == "" {
		return nil, errors.New("please set $BROWSER to a web launcher")
	}
	return shellquote.Split(browser)
}

// 根据操作系统，返回默认browser command
func searchBrowserLauncher() (browser string) {
	switch runtime.GOOS {
	case "darwin":
		browser = "open"
	case "windows":
		browser = "cmd /c start"
	case "linux":
		browser = "xdg-open"
	default:
		browser = ""
	}
	return browser
}
```

最后通过`go build -o git-open`，即可通过git-open，在浏览器中打开当前项目的远端页面。

