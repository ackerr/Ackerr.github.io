---
layout: post
title: Golang中 println 与 fmt.Println 有什么区别 
date: 2021-07-06 23:46:49
tags:
    - golang
---

## 前言

在之前写的一个[命令行工具](https://github.com/Ackerr/lab/)，其中一个功能点是选择某个项目，并切换至该项目目录 （对应CLI的lab cs命令）。因为程序中不能直接修改shell当前路径，只好在命令执行后输出内容，再与shell内置命令搭配使用来实现。

伪代码如下：

```go
package main

import "fmt"

func main() {
	path := GetPath()
	fmt.Println(path)
}

func GetPath() string {
	// do something
	return "/usr/local/bin/"
}
```

正常情况下，执行`cd $(go run main.go)`后，shell路径会切换到`/usr/local/bin`。

但在一次修改中，年轻的我以为`println`与`fmt.Println`的输出是等价的，都是写入标准输出，但`println`是内置函数，不需要额外导入fmt包，就把项目中使用`fmt.Println`的地方都替换成`println`，从而[埋下祸根](https://github.com/Ackerr/lab/commit/a6611f05addc7a3d2caa168c9a90847bef249125#diff-e0f0eb674c3c8261dd55ff4e687ee203e54b8ccbc4d4678d3930b9fd4ea18d68R60)。

今早在本地更新了该命令行后，发现`lab cs`命令不好使了，虽然能在终端下看到该命令输出路径，但结合cd使用却没有效果，然而前一个版本，没问题的，间接定位到BUG是到改println的pr引入的。

## 1.输出位置不同

经紫月提醒，发现使用`cd $(lab cs 2&>1)`能正确切换路径了，那么原因很明确了，`println`把内容输出到标准错误中了。

查看`fmt.Println`的源码，注释和代码中都很明显：**writes to standard output**，内容输出到了os.Stdout，也就是标准输出。

```go
// Println formats using the default formats for its operands and writes to standard output.
// Spaces are always added between operands and a newline is appended.
// It returns the number of bytes written and any write error encountered.
func Println(a ...interface{}) (n int, err error) {
	return Fprintln(os.Stdout, a...)
}
```

再来看看`println`，因为是内置函数，这里只写了函数说明，不过在注释中，可以找到这句：**writes the result to standard error**。说明`println/print`确实是输出到stderr，也就是标准错误输出。

```go
// The println built-in function formats its arguments in an
// implementation-specific way and writes the result to standard error.
// Spaces are always added between arguments and a newline is appended.
// Println is useful for bootstrapping and debugging; it is not guaranteed
// to stay in the language.
func println(args ...Type)
```

我们可以在go源码的 `runtime/print.go`找到`println`的具体实现。代码如下：

```go
func printnl() {
	printstring("\n")
}

func printstring(s string) {
	gwrite(bytes(s))
}

// write to goroutine-local buffer if diverting output,
// or else standard error.
func gwrite(b []byte) {
	if len(b) == 0 {
		return
	}
	recordForPanic(b)
	gp := getg()
	// Don't use the writebuf if gp.m is dying. We want anything
	// written through gwrite to appear in the terminal rather
	// than be written to in some buffer, if we're in a panicking state.
	// Note that we can't just clear writebuf in the gp.m.dying case
	// because a panic isn't allowed to have any write barriers.
	if gp == nil || gp.writebuf == nil || gp.m.dying > 0 {
		writeErr(b)
		return
	}
	n := copy(gp.writebuf[len(gp.writebuf):cap(gp.writebuf)], b)
	gp.writebuf = gp.writebuf[:len(gp.writebuf)+n]
}
```

在`gwrite`函数中，这里主要看下`writeErr`

```go
func writeErr(b []byte) {
	write(2, unsafe.Pointer(&b[0]), int32(len(b)))
}

func write(fd uintptr, p unsafe.Pointer, n int32) int32 {
	return write1(fd, p, n)
}

func write1(fd uintptr, p unsafe.Pointer, n int32) int32
```

可以看到writeErr传递了unitptr为2的fd参数，最终在write1处进行系统调用。而标准错误输出的文件描述符刚好是2。

> 在unix中，stdin，stdout，stderr三者对应三个文件描述符，分别是0，1，2

## 2.函数定义不同

平时都是直接使用`println("?")` 或 `fmt.Println("?")` ，不过两者的函数定义也有些不同。

```go
// println
func println(args ...Type)

// fmt.Println
func Println(a ...interface{}) (n int, err error)
```

**虽然两者都接受多个传参，但只有`fmt.Println`是具有返回值的**，其中第一位返回输出内容的字节数，对于`Println`还需要加上末尾换行符`\n`的一字节。

除此之外，`println/print` 不接受数组和结构体参数。

## 3.输出格式不同

如果实参实现了`String()或Error()`方法时，在调用`fmt.Println`打印该参数时，会调用这两个方法，而内置的`println/print`则不会使用。


### 参考

1. https://github.com/golang/go/blob/master/src/runtime/print.go#L86
2. https://gfw.go101.org/article/unofficial-faq.html#print-builtin-fmt-log
3. https://www.zhihu.com/question/335186436
