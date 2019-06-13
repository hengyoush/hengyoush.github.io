---
layout: post
title:  "JVM方法执行探究"
date:   2019-06-12 19:47:00 +0700
categories: [jvm]
---

首先看一个小例子，有两个方法如下：
```java
void foo(Object o, Object...objs);
void foo(String s, Object obj, Object...objs);

foo(null, 1); // 调用2
foo(null, 2, 1); // 调用2
foo(null, new Object[]{1}); // 避开可变参数，调用1
```

这是由于`null`相比`String`和`Object`，与String更加接近。故重载时选择了第二个方法。

## 重载的规则
重载方法选取的规则如下：
1. 不考虑对基本类型装箱拆箱的情况下选取重载方法。
2. 如果第一步没有选到，那么运行装箱拆箱的情况下不允许变长参数的情况下选取重载方法。
3. 如果第二步没有找到，那么在允许装箱拆箱的情况下而且允许变长参数的情况下选取重载方法。

此外如果编译器在同一个阶段中找到了多个匹配的方法，那么它选择其中一个最为贴切的方法，决定贴切程度的关键就是
类的继承关系.

## JVM如何识别方法
类名,方法名,方法描述符(method descriptor):由方法的参数类型和返回类型组成.如果出现名字相同且方法描述符也相同的方法,JVM在验证阶段报错.
**JVM**对于重写的判断是基于名字相同且方法描述符也相同的情况.
对于 Java 语言中重写而 Java 虚拟机中非重写的情况,编译器会生成桥接方法.

## JVM的静态绑定和动态绑定
重载的区分在编译阶段已经完成, 是为静态绑定.重写被称为动态绑定.

字节码中与方法调用相关的指令有五种.
1. `invokestatic`: 调用静态方法
2. `invokespecial`: 调用私有实例方法、构造器、默认方法等
3. `invokevirtual`: 调用非私有实例方法
4. `invokeinterface`: 调用接口方法
5. `invokedynamic`: 调用动态方法

## 方法的符号引用
看如下代码:
```java
public class Test {
    public static void main(String[] args) {
        A a = new B();
        a.isA(); // interfaceMethodRef
        B b = (B) a;
        b.S(); // MethodRef
    }

    interface A {
        void isA();
    }

    static class B implements A {

        public void S() {}

        @Override
        public void isA() {

        }
    }
}
```

使用`javap -v Test.class`得到该类的字节码, 下面是`main`方法的片段

```java
...
9: invokeinterface #4,  1            // InterfaceMethod com/aliware/tianchi/Test$A.isA:()V
...
20: invokevirtual #5                  // Method com/aliware/tianchi/Test$B.S:()V
...
```
可以看到`isA`方法是`InterfaceMethod`, 而`S`方法是`Method`, 这两个方法的符号引用在解析的时候是具有不同的行为的.

对于`Method`, 有如下解析步骤:
1. 先在本类中查找是否存在相同方法名和方法描述符的方法(这一步会发生*静态方法隐藏*)
2. 如果上一步没有找到, 那么向上回溯, 直到Object重复1.
3. 如歌没有找到, 在该类中实现的接口或间接实现的接口中寻找, 这一步寻找的目标方法必须是非静态的(接口中定义的静态方法必须由接口来调用, 否则java编译器会报错),非私有的.这一步有可能向上回溯.

对于`InterfaceMethod`, 有如下解析步骤:
1. 在I中查找相同方法名和方法描述符的方法
2. 在Object中的公有实例方法中寻找
3. 在I的超接口中寻找

经过如上的解析步骤, 符号引用被解析为实际引用, 对于静态绑定的方法而言, 实际引用是一个方法指针, 对于动态绑定而言, 实际引用是指向方法表条目的索引.

