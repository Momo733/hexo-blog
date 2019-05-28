---
title: golang编译
date: 2019-05-28 15:56:47
tags:
 - golang build
 - ldflags 参数
---

在golang中有许多种的编译，下面讲一下我自己在编译过程中遇到的一些坑。

golang的编译命令非常的简单，在对应的main文件下直接``go build``,但是如果我们对应编译的程序在不同的环境下使用，例如数据库的DSN，端口等重要数据是不可能放在代码中记录的，这个时候就需要动态的设置这些参数，但是golang是静态的语言，怎么办呢，但是golang提供了两种方法可以从外部获取到一些参数的信息。

## 1.从环境变量中获取

golang提供``os.GetEnv()``方法,可以直接从编译的本机的环境变量中直接获取到对应的参数信息。
```
packeg main

func main(
    dsn:=os.GetEnv("DSN")
    print(dsn)
)
```
这种获取的是字符串的参数，需要你提前配置在机器的环境变量之中。
但是总感觉这种办法虽然方便，但还是不太灵活，其实是因为golang的编译工具里面就有相关的方法来解决这个问题。

## 2.从编译参数中获取

使用golang的编译工具``ldflags``，使用这种的编译放入参数的方式，可以解决灵活性的问题，但是定义的参数必须是未初始化的字符串。
```
-X importpath.name=value
	Set the value of the string variable in importpath named name to value.
	This is only effective if the variable is declared in the source code either uninitialized
	or initialized to a constant string expression. -X will not work if the initializer makes
	a function call or refers to other variables.
	Note that before Go 1.5 this option took two separate arguments.
```

这是官方的原文档，不仅不能定义已经初始化的字符串，并且不能对函数和已经引用的参数传值，这将会不起作用。

```
packeg main

var  dsn string
var date string

func main(
    print(dsn,data) //root:123@tcp(8.8.8.8:3306)/serviceSun Feb 12 00:13:27 CST 2019
)
```

使用命令如下：

```
go build -ldflags "-X main.dsn=root:123@tcp(8.8.8.8:3306)/service -X 'main.date=`date`'"
```

使用``go build `` 可以直接输入参数，并且可以使用一些外部定义好的变量，例如``date``,``go version``等。