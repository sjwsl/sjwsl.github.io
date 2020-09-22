---
layout: post
title: Effective STL Reading Notes
categories: 
    - Reading Notes
tag:
    - C++
    - STL
---

*Effective STL* Reading Notes

## 引言

STL 包括迭代器、容器、函数对象和算法。

迭代器分类
- 输入迭代器（input iterator）：只读，在每个被遍历到的位置只能读一次
- 输出迭代器（output iterator）：只写，在每个被遍历到的位置只能写一次
- 前向迭代器（forward iterator）：可重复读写，不支持 --
- 双向迭代器（bidirectional iterator）：可重复读写，支持 --
- 随机访问迭代器（random access iterator）：在双向迭代器的基础上，支持迭代器算术

容器分类
- 标准序列容器：vector string deque list
- 标准关联容器：set multiset map multimap
- 非标准序列容器： slist（单向链表） rope
- 非标准关联容器：hash_set hash_multiset hash_map hash_multimap
- 标准的非 STL 容器：数组 bitset valarray stack queue priority_queue

所有重载了`()`的类都称为一个**函数子类（functor class）**，这些类的实例称为**函数对象（function object）**或**函数子（functor）**。

当我们讨论 STL 时，要考虑 STL 的不同实现，以及不同编译器对 STL 的不同支持。

## 容器

### 第1条：慎重选择容器类型

- 是否有序
- 所需迭代器类型
- 是否必须为标准 C++ 的一部分
- 数据的布局是否要兼容 C
- 是否要避免使用引用计数
- 是否需要事务语义

按照数据存储方式，容器可以分为**连续内存（contiguous-memory）**和**基于节点（node-based）**的。
- 连续内存：vector string deque rope
  元素被存储在一块或多块动态分配的内存中，每块内存有多个元素。当有元素删除或者插入，同一内存块的其他元素要移动。这种移动影响到效率和异常安全性。
- 基于节点：list 关联容器
  每一个动态分配的内存块中只存放一个元素，元素的插入和删除只影响指向节点的指针，而不影响节点本身的内容

### 第2条：不要试图编写独立于容器类型的代码

不同的 STL 容器支持的操作差别很大，因此对容器的泛化一般来说是没有意义的。

考虑到有时候不可避免地要更换容器类型，可以简单地用 `typedef` 来进行词法（lexical）层面上的封装。

### 第3条：确保容器中的对象副本正确且高效

不要尝试把派生类插入到基类容器中，否则在复制的过程中会导致**剥离（slicing）**，丢失派生类特有的所有信息。如果想这么做，应该构造一个存放智能指针的容器。

### 第4条：用`empty()`而不是`size()==0`

只是因为一些 list 的实现中 `size()` 耗费线性时间

### 第5条：优先使用区间函数而不是多次使用单元素函数

给定两个 vector `v1` 和 `v2`，让 `v1` 等于 `v2` 的后半部分。

最差的循环写法
```c++
v1.clear();
for (auto iter = v2.begin() + v2.size() / 2; iter != v2.end(); ++iter) {
    v1.push_back(*iter);
}
```

用 `std::copy` 的写法，本质上其实也是循环
```c++
v1.clear();
std::copy(v2.begin() + v2.size() / 2, v2.end(), back_inserter(v1));
```

几乎所有的利用插入迭代器（`inserter`, `front_inserter`, `back_inserter`）的 `std::copy` 都可以用区间版本的 `insert` 代替。

用区间函数的写法 
```c++
v1.assign(v2.begin() + v2.size() / 2, v2.end());
```

使用区间函数可以使代码更清晰，且很多时候效率更高。

常用的区间函数（`iterator` 指随机访问迭代器，`InputIterator` 接受除输出迭代器外的所有迭代器）
- 区间构造函数
```c++
container::container(InputIterator begin, InputIterator end);
```
- 区间插入

序列容器
```c++
void container::insert(iterator position, InputIterator begin, InputIterator end);
```
关联容器的版本不用指定插入位置
```c++
void container::insert(InputIterator begin, InputIterator end);
```
- 区间删除

序列容器，返回指向被删除元素的后一个元素
```c++
iterator container::erase(iterator begin, iterator end);
```
关联容器，标准提到如果如果也要返回后一个元素会带来不可接受的性能负担，因此为 void 函数=
```c++
void container::erase(iterator begin, iterator end);
```
- 区间赋值
```c++
void container::assign(InputIterator begin, InputIterator end);
```

### 第6条：小心C++的分析机制

```c++
ifstream dataFile("ints.data");
list<int> data(istream_iterator<int>(dataFile), istream_iterator<int>();
```

这段代码可以通过编译，但是什么也不会做。编译器会尽可能把语句解释为函数声明，因此上面本来想创建 list 的代码被解释为函数的声明。函数的返回值类型是 `list<int>`，第一个参数类型是 `istream_iteratot<int>`，名称是 dataFile。第二个参数类型是一个签名为`istream_iterator()`函数指针，没有名称。

解决方法是避免使用匿名对象
```c++
istream_iterator<int> begin(dataFile);
istream_iterator<int> end;
list<int> data(begin, end);
```

### 第7条：容器析构时不负责 delete 存放的指针

C++11 后，这条可以简单地用 `unique_ptr` 或 `shared_ptr` 避免。

### 第8条：不要创建 `auto_ptr` 的容器

这条也是 C++11 之前的历史遗留问题。

### 第9条：选择删除元素的方法

删除容器中所有值为 1963 的元素。

对于标准序列容器，最简单的写法是 erase-remove
```c++
c.erase(remove(c.begin(), c.end(), 1963), c.end());
```

对于 list，还有更简单的写法
```c++
c.remove(1963);
```

对于标准关联容器，没有 `remove` 成员函数
```c++
c.erase(1963)
```

有 ``BadValue(int x)``，删除容器中所有的 bad value。

对于标准序列容器，简单地使用 `remove_if` 替换 `remove` 即可
```c++
c.erase(remove_if(c.begin(), c.end(), BadValue), c.end());
c.remove_if(BadValue); // list
```

对于标准关联容器，可以用循环删除。下面是错误的写法
```c++
for (auto iter = c.begin(); iter != c.end(); ++iter) {
    if (BadValue(*iter)) c.erase(iter);
}
```

错误的原因是 `c.erase(iter)` 后 `iter` 的值就无效了，`++iter` 就不会返回想要的结果。解决方案是在删除之前递增
```c++
for (auto iter = c.begin(); iter != c.end(); ) {
    if (BadValue(*iter)) c.erase(iter++);
    else iter++;
}
```

如果要在删除的同时做一些其他事情（比如写日志），序列容器就没法用 erase-remove 的写法了。之前的两种循环写法对于序列容器都是错误的，因为序列容器调用 `erase` 不仅会使删除的迭代器失效，这个位置之后的所有迭代器都会失效。解决方案是利用 `erase` 的返回值。
```c++
for (auto iter = c.begin(); iter != c.end(); ) {
    if (BadValue(*iter)) {
        // do something
        iter = c.erase(iter);
    }
    else iter++;
}
```

### 第10条：分配子（allocator）的约定和限制

### 第11条：自定义分配子的合理用法

### 第12条：容器的线程安全性

STL 的所有容器都只保证多线程同时读是线程安全的。

## vector 和 string

### 第13条：优先使用 vector 和 string 而不是内建数组

使用 vector 和 string 可以免除内存管理的烦恼

有一种情况下要注意 string 的使用。有的 string 实现用到了引用计数，在多线程下同步控制的消耗很大，这种情况下可以用 `vector<char>` 代替。

### 第14条：使用 reserve 来避免不必要的重新分配

vector 和 string 的容量会自动增长。当需要更多空间时，会重新分配一块大小为当前容量某个倍数的内存（大多数实现为 2 倍），然后把容器所有元素复制到新的内存，最后析构原内存中的对象并释放旧内存。需要注意的是，重新分配后容器原来所有的指针、迭代器、引用都会失效。