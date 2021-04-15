---
layout: post
title:  "c++模版笔记"
date:   2020-09-26 22:09:00 +0700
categories: [c++]
---

## 基础

### 类模版声明与定义
header中的类声明示例如下:
```cpp
template <typename T>
class Grid {
    ...
    Grid(const Grid& src) = default;
    Grid<T>& operator=(const Grid& rhs) = default;
    
    const std::optional<T>& at(size_t x, size_t y) const;
}
```

cpp中定义如下:
```cpp
template <typename T>
const std::optional<T>& Grid<T>::at(size_t x, size_t y) const {
    ...
}
```

### 方法模版声明与定义
因为Grid<T>与Grid<E>不是一个类型,所以通常情况下它们之间无法相互赋值.
然而通过定义新的拷贝和赋值函数可以实现这一点.  
声明方法:
```cpp
template <typename T>
class Grid
{
    public:
        // Omitted for brevity

        template <typename E>
        Grid(const Grid<E>& src);

        template <typename E>
        Grid<T>& operator=(const Grid<E>& rhs);

        void swap(Grid& other) noexcept;

        // Omitted for brevity
};
```
我们在每个方法上都加了新的模版声明`template <typename E>`以此来代表与T是不同的类型.  
另外一种声明方式:
```cpp
template <typename T>
template <typename E>
Grid<T>::Grid(const Grid<E>& src)
    : Grid(src.getWidth(), src.getHeight())
{
    // The ctor-initializer of this constructor delegates first to the
    // non-copy constructor to allocate the proper amount of memory.

    // The next step is to copy the data.
    for (size_t i = 0; i < mWidth; i++) {
        for (size_t j = 0; j < mHeight; j++) {
            mCells[i][j] = src.at(i, j);
        }
    }
}
```
注意`template <typename E>`必须在`template <typename T>`的下面.  
而且在这样的copy 函数和赋值函数中使用到的其他成员函数必须是public的.

copy和赋值函数定义如下:
```cpp
template <typename T>
template <typename E>
Grid<T>& Grid<T>::operator=(const Grid<E>& rhs)
{
    // no need to check for self-assignment because this version of
    // assignment is never called when T and E are the same

    // Copy-and-swap idiom
    Grid<T> temp(rhs); // Do all the work in a temporary instance
    swap(temp); // Commit the work with only non-throwing operations
    return *this;
}
```

## 进阶

### 选择性初始化 (Selective Instantiation)
对于虚函数, 编译器会自动生成相关代码.  
而对于非虚函数, 如果不显式调用, 编译器不会自动生成代码.

### 将template代码的声明与定义分开
通常我们只需要include一个class的header就可以调用它的函数了, 具体的定义在link时得到,  
但是对于template class就不可以,  因为它要在编译时确定所有, 所以必须能够在编译时就将方法定义等
include进来, 但是为了保持良好的代码结构, 我们需要一种方法能够保证定义与声明分离.

使用这样一种方法, 将方法定义分离到另外一个header中,然后在声明的header里去include实现的header,
注意这个include需要放在文件的末尾.
```cpp
template <typename T>
class Grid
{
    // Class definition omitted for brevity
}
#include "GridDefinitions.h"
```
### template 参数
template定义中可以不仅仅包含type参数, 也可以包含其他参数, 比如如下的定义:
```cpp
template <typename T, size_t MAX_WIDTH = 10>
class Grid {
    ...
}
```
这里甚至可以给出一个默认值.
注意`Grid<T, 2>`和`Grid<T, 3>`不是一个类型,不可相互赋值.而且在初始化模版的时候,
不能使用变量初始化比如`Grid<T, width>`是非法的.

### template根据构造器进行类型推断
原先可能要这样写:
```cpp
std::pair<int, double> pair1(1, 2.3);
```
现在只要这样写:
```cpp
std::pair pair3(1, 2.3);
```

### template特别定制款
可以为某一个class专门定制实现, 如下:
```cpp
// 必须包含原template class声明
#include "Grid.h"

// 必须要有!
template <>
class Grid<const char*>
{
    public:
        explicit Grid(size_t width = kDefaultWidth,
            size_t height = kDefaultHeight);
        virtual ~Grid() = default;

        // Explicitly default a copy constructor and assignment operator.
        Grid(const Grid& src) = default;
        Grid<const char*>& operator=(const Grid& rhs) = default;

        // Explicitly default a move constructor and assignment operator.
        Grid(Grid&& src) = default;
        Grid<const char*>& operator=(Grid&& rhs) = default;

        std::optional<std::string>& at(size_t x, size_t y);
        const std::optional<std::string>& at(size_t x, size_t y) const;

        size_t getHeight() const { return mHeight; }
        size_t getWidth() const { return mWidth; }

        static const size_t kDefaultWidth = 10;
        static const size_t kDefaultHeight = 10;

    private:
        void verifyCoordinate(size_t x, size_t y) const;

        std::vector<std::vector<std::optional<std::string>>> mCells;
        size_t mWidth, mHeight;
};
```
可以看到这种定制款可以完全和template class完全不一样.

### 继承模版类
和继承普通类一样, 继承模版类的声明也是相当自然的.
```cpp
#include "Grid.h"

template <typename T>
class GameBoard : public Grid<T>
{
    public:
        explicit GameBoard(size_t width = Grid<T>::kDefaultWidth,
            size_t height = Grid<T>::kDefaultHeight);
        void move(size_t xSrc, size_t ySrc, size_t xDest, size_t yDest);
};
```
GameBoard继承了Grid的所有方法.

需要注意:使用泛型继承时,子类使用父类的方法时必须在方法前指定父类的名字:如Grid<T>::

```cpp
template <typename T>
GameBoard<T>::GameBoard(size_t width, size_t height)
    : Grid<T>(width, height)
{
}

template <typename T>
void GameBoard<T>::move(size_t xSrc, size_t ySrc, size_t xDest, size_t yDest)
{
    // 注意这里, 调用at方法时显式指名了父类
    Grid<T>::at(xDest, yDest) = std::move(Grid<T>::at(xSrc, ySrc));
    Grid<T>::at(xSrc, ySrc).reset();  // Reset source cell
    // Or:
    // this->at(xDest, yDest) = std::move(this->at(xSrc, ySrc));
    // this->at(xSrc, ySrc).reset();
}
```

## 题外话
1. 一旦显式定义了析构函数,那么就必须显式定义copy构造函数和copy赋值函数,否则编译器默认将其禁用.