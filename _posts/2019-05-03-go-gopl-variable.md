---
layout: post
title:  "Golang学习:变量命名与声明"
date:   2019-05-03 12:32:00 +0700
categories: [golang]
---
## 命名
1. 一个名字必须以一个字母或者下划线开头.
2. 在包内定义的变量如果开头字母是大写的话那么它是包外可见的,如果是小写的那么它只能被包内访问.
3. 越小的作用域使用尽量简短的变量命名.

#### 关键字
Go语言定义了一些关键字,这些关键字不能作为变量的命名
```
break      default       func     interface   select
case       defer         go       map         struct
chan       else          goto     package     switch
const      fallthrough   if       range       type
continue   for           import   return      var
```

还有一些预定义的名字,作为内建数据类型或函数,可以在定义中重新使用.
```
内建常量: true false iota nil

内建类型: int int8 int16 int32 int64
          uint uint8 uint16 uint32 uint64 uintptr
          float32 float64 complex128 complex64
          bool byte rune string error

内建函数: make len cap new append copy close delete
          complex real imag
          panic recover
```
## 声明
有四个关键字可以声明变量,分别是`var` `const` `type` `func`四个,下面来分别介绍.

### var关键字
声明格式:
```go
var 变量名称 类型 = 表达式
```
其中`类型`和`= 表达式`可以省略其中一个.

如果省略的是表达式,那么该变量会根据默认的零值进行初始化(不像Java一样如果在方法内没有赋初始值就无法使用)

### 简短变量声明
声明格式
```go
变量名 := 表达式
```
例如:
```go
i := 100
i, j := 1, 0 // 多个变量赋值
f, err := os.Open(name) // 利用函数的返回值进行赋值
```

## 指针
一个指针的值是另一个变量的地址.可以通过`&x`来获取变量x的指针.
通过`*p`来对p指针指向的变量的值进行操作.
例如:
```go
i := 1
p := &i // 声明p为变量x的指针
fmt.print(*p) // 1
*p = 2
fmt.print(*p) // 2
```

除了通过`&`运算符获取地址之外,还可以通过new函数获取地址.如下:
```go
p := new(int) // 声明一个匿名的int变量,p为匿名变量的指针
*p = 2 // 通过指针p改变匿名变量的值
```