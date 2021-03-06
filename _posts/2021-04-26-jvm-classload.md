---
layout: post
title:  "JVM笔记-类加载笔记"
date:   2021-04-26 23:20:00 +0700
categories: [jvm]
---

## 作用
将Class文件转化为JVM能够直接使用的形式存储在方法区中.

## 时机
JVM规定以下六种情况是触发类加载的场景:
1. 使用new、getstatic、putstatic、invokestatic指令
2. 使用java.lang.reflect包进行反射调用的时候
3. 初始化类时如果父类没有初始化则需要初始化父类
4. 虚拟机启动时的主类
5. 使用JDK7新加入的动态语言支持时,如果一个MethodHandle实例最后解析的结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄,且该方法句柄对应的类没有初始化则需要触发初始化.
6. 定义JDK8的默认方法时,当有该接口的实现类发生了初始化.那么该接口要在其之前被初始化.

### 主动引用
即JVM定义的六种场景

### 被动引用
除六种场景之外的引用方式.
1. 比如通过子类引用父类的静态变量的方式.
2. 引用类中定义的常量
3. 引用数组的时候不会引发数组元素类型的初始化.

## 过程

### 加载
通过类的全限定名获取二进制字节,读取字节将其中的信息存储到方法区内部并且生成一个Class对象作为后续类数据的访问入口.

### 验证
校验Class文件的字节流是否符合JVM规定的格式, 校验字节码逻辑是否合法.

### 准备
为类的静态变量分配空间

### 解析
将符号引用替换为直接引用

#### 什么是符号引用、直接引用
符号引用: 一组符号用于描述所引用的目标.

直接引用: 可以直接定位到引用目标, 可以是指针、偏移量或者是一个句柄.

### 初始化
执行类的clinit方法的过程.初始化是线程安全的,只有一个线程可以初始化.

## 类加载器
通过一个类的全限定名获取到类的二进制字节流这个动作由类加载器实现.

### 类加载器的隔离特性
只有同一个类加载器加载的类创建的对象的比较才有意义.

### 双亲委派模型
Bootstrap ClassLoader
Extension ClassLoader
Application ClassLoader
三层类加载器.

工作流程: 当一个类加载器收到加载类的请求时, 它首先将该请求委派给它的父加载器, 如果父加载器加载
不到的话, 才自己加载.
这样做的目的: 安全性,防止一些重要的基础类被其他恶意jar包定义.
比方说, 我在demo.jar里定义了这样一个类:  
```java
public class String {
   public String() {
      // 做些坏事情
   }
}
```
然后你引用了我的jar包,使用了我的代码, 我在代码的某处加载了我自己写的这个String类, 然后你再使用String类的时候
使用的就是我加了点“料”的String类了.  
而加入双亲委派模型之后, 所有类都先被委托给父加载器加载, 这样对于java的核心类就只能由Bootstrap ClassLoader来加载,而Bootstrap ClassLoader是一定从特定的jar包中查找类(比如rt.jar),不会出现上面的情况.








