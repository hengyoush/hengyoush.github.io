---
layout: post
title:  "c++ cast笔记"
date:   2020-09-27 21:36:00 +0700
categories: [c++]
---

## const cast
const_cast可以给一个不是const的变量增加const属性, 也可以把一个变量的cast给去掉.
例子:
```cpp
extern void ThirdPartyLibraryMethod(char* str);

void f(const char* str)
{
    ThirdPartyLibraryMethod(const_cast<char*>(str));
}
```
要注意, ThirdPartyLibraryMethod仍然不可以修改参数的值.

下面是std::as_const的例子,它可以把一个非const变量变为const变量.
```cpp
std::string str = "C++";
const std::string& constStr = std::as_const(str);
```


## static cast
主要用来进行c++本身就支持的转换, 比如int到double的转换.
也可以用来进行用户自定义的转换, 比如class A的构造器接收一个B对象,那就可以使用static cast
进行B到A的转换.

除此之外, static cast还可以进行**继承关系**中的向下转型(无关的类型转换不可以使用static_cast).
例子:
```cpp
class Base {
public:
    virtual ~Base() = default;    
}

class Derived : public Base {
public:
    virtual ~Derived() = default;
}

int main() {
    Base* b;
    Derived* d = new Derived;
    b = d;
    d = static_cast<Derived*>(b);
    
    Derived derived;
    Base& br = derived;
    Derived& dr = static_cast<Derived&>(br);
}
```
static_cast作用在继承关系上的转型只可以用于**指针**和**引用**.  
但是这种转型在某些情况下是不安全的, 比如:
```cpp
Base* b = new Base{};
Derived* d = static_cast<Derived*>(b);
```
这段代码可以编译并执行且不报错,但是却隐藏了极大的未知内存数据被覆盖的风险.

## reinterpret cast
可以进行一些static cast不允许的转换.比如将两个毫不相关类型的引用/指针互转.
比如转为void*.
例子:
```cpp
class A {};
class B {};

int main() {
    A a;
    B b;
    A* ap = &a;
    B* bp = &b;
    
    // 隐式转换
    void* p = ap;
    // 此时需要使用reinterpret_cast显式转换
    ap = reinterpret_cast<A*>(p);
    
    // 对于毫不相关类型的引用转换也可以使用reinterpret_cast
    A& ar = a;
    B& br = reinterpret_cast<B&>(ar);
    return 0;
}
```

## dynamic cast
提供了运行期的类型检查, 如果不是可以互转的类型那么会:
1. 返回一个nullptr(指针的情况)
2. 抛出一个std::bad_cast的异常.(引用的情况)  
例子:
```cpp
class Base {
public:
    virtual ~Base() = default;    
}

class Derived : public Base {
public:
    virtual ~Derived() = default;
}

int main() {
    // 正常的情况
    Derived* d = new Derived();
    Base* b = d
    d = dynamic_cast<Derived*>(b);
    
    // 错误的情况
    Base b;
    Base& br = b;
    try {
        Derived& dr = dynamic_cast<Derived&>(br);
    } catch (std::bad_cast&) {
        cout << "bad cast" << endl;
    }
    
    return 0;
}
```
要注意, 使用dynamic_cast必须保证相关类型有virtual方法,因为类型信息是存储在
对象的vtable中的.
