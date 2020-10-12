---
# layout: post
title: Effective STL Reading Notes
categories:
  - Reading Notes
tag:
  - C++
  - STL
---

50 specific ways to improve your use of the Standard Template Library

## 引言

STL 包括迭代器、容器、函数对象和算法。

迭代器分类

- 输入迭代器（input iterator）：只读，在每个被遍历到的位置只能读一次
- 输出迭代器（output iterator）：只写，在每个被遍历到的位置只能写一次
- 前向迭代器（forward iterator）：可重复读写，不支持 --
- 双向迭代器（bidirectional iterator）：可重复读写，支持 --
- 随机访问迭代器（random access iterator）：在双向迭代器的基础上，支持迭代器算术

容器分类

- 标准序列容器：`vector` `string` `deque` `list`
- 标准关联容器：`set` `multiset` `map` `multimap`
- 非标准序列容器： `slist`（单向链表） `rope`
- 非标准关联容器：`hash_set` `hash_multiset` `hash_map` `hash_multimap`
- 标准的非 STL 容器：数组 `bitset` `valarray` `stack` `queue` `priority_queue`

所有重载了`()`的类都称为一个**函数子类（functor class）**，这些类的实例称为**函数对象（function object）**或**函数子（functor）**。

当我们讨论 STL 时，要考虑 STL 的不同实现，以及不同编译器对 STL 的不同支持。

## 容器

### 第 1 条：慎重选择容器类型

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

### 第 2 条：不要试图编写独立于容器类型的代码

不同的 STL 容器支持的操作差别很大，因此对容器的泛化一般来说是没有意义的。

考虑到有时候不可避免地要更换容器类型，可以简单地用 `typedef` 来进行词法（lexical）层面上的封装。

### 第 3 条：确保容器中的对象副本正确且高效

不要尝试把派生类插入到基类容器中，否则在复制的过程中会导致**剥离（slicing）**，丢失派生类特有的所有信息。如果想这么做，应该构造一个存放智能指针的容器。

### 第 4 条：用 `empty()` 而不是 `size()==0`

只是因为一些 list 的实现中 `size()` 耗费线性时间

### 第 5 条：优先使用区间函数而不是多次使用单元素函数

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

**常用的区间函数（`iterator` 指随机访问迭代器，`InputIterator` 接受除输出迭代器外的所有迭代器）**

**区间构造函数**

```c++
container::container(InputIterator begin, InputIterator end);
```

**区间插入**

序列容器

```c++
void container::insert(iterator position, InputIterator begin, InputIterator end);
```

关联容器的版本不用指定插入位置

```c++
void container::insert(InputIterator begin, InputIterator end);
```

**区间删除**

序列容器，返回指向被删除元素的后一个元素

```c++
iterator container::erase(iterator begin, iterator end);
```

关联容器，标准提到如果如果也要返回后一个元素会带来不可接受的性能负担，因此为 void 函数=

```c++
void container::erase(iterator begin, iterator end);
```

**区间赋值**

```c++
void container::assign(InputIterator begin, InputIterator end);
```

### 第 6 条：小心 C++ 的分析机制

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

### 第 7 条：容器析构时不负责 `delete` 存放的指针

C++11 后，这条可以简单地用 `unique_ptr` 或 `shared_ptr` 避免。

### 第 8 条：不要创建 `auto_ptr` 的容器

这条也是 C++11 之前的历史遗留问题。

### 第 9 条：选择删除元素的方法

删除容器中所有值为 1963 的元素。对于标准序列容器，最简单的写法是 erase-remove

```c++
c.erase(remove(c.begin(), c.end(), 1963), c.end());
```

对于 list，还有更简单的写法

```c++
c.remove(1963);
```

对于标准关联容器，没有 `remove` 成员函数

```c++
c.erase(1963);
```

有 `BadValue(int x)`，删除容器中所有的 bad value。对于标准序列容器，简单地使用 `remove_if` 替换 `remove` 即可

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

### 第 10 条：分配子（allocator）的约定和限制

### 第 11 条：自定义分配子的合理用法

### 第 12 条：容器的线程安全性

STL 的所有容器都只保证多线程同时读是线程安全的。

## `vector` 和 `string`

### 第 13 条：优先使用 `vector` 和 `string` 而不是内建数组

使用 vector 和 string 可以免除内存管理的烦恼

有一种情况下要注意 string 的使用。有的 string 实现用到了引用计数，在多线程下同步控制的消耗很大，这种情况下可以用 `vector<char>` 代替。

### 第 14 条：使用 `reserve` 来避免不必要的重新分配

vector 和 string 的容量会自动增长。当需要更多空间时，会重新分配一块大小为当前容量某个倍数的内存（大多数实现为 2 倍），然后把容器所有元素复制到新的内存，最后析构原内存中的对象并释放旧内存。需要注意的是，重新分配后容器原来所有的指针、迭代器、引用都会失效。

### 第 15 条：`string` 的实现多样性

### 第 16 条：`vector` 和 `string` 和 C 风格的兼容

对于一个 `vector v`，因为内存是连续的， `&v[0]` 就可以当作 C 风格数组使用。但是要注意判断 `v.empty()==false`。`v.begin()` 返回的是迭代器，并不总是可以当成指针（见第 50 条）。

`vector` 可以通过 `&v[0]` 向 C 风格数组读写，其他 STL 容器和数组的交互可以通过 `vector` 来完成，因为只有 `vector` 保证内存是连续的。

### 第 17 条：使用 `swap` 去除多余的容量

当用 `erase` 等函数从 `vector` 或 `string` 删除元素后，它的容量并不会减小。用 `swap` 可以释放掉浪费的内存

```c++
vector<int>(v).swap(v);
```

这句话构造一个临时的 `vector`，调用复制构造函数，然后和 `v` 交换。这种方法基于复制构造函数只会分配必要的内存。

同样，`clear` 可能并不会释放内存，可以用 `swap` 方法清空 `vector` 并释放内存

```c++
vector<>().swap(v)
```

### 第 18 条：不要使用 `vector<bool>`

`vector<bool>` 并不是 STL 容器，它使用类似 `bitset` 的方法存储 `bool` 值，所以无法获得其中的一个值的指针（因为它只是在内存中的一个二进制位）。

用 `deque<bool` 或者 `bitset` 替代 `vector<bool>`。

## 关联容器

### 第 19 条：区别相同（equality）和等价（equivalence）

关联容器经常需要判断两个值是否相同，然而判断标准是不一样的。例如 `find` 和 `insert` 都要判断 `set` 中是否有给定元素，然而 `find` 对“相同”的定义是相等，是以 `==` 为基础的，而 `insert` 却是以等价为标准，是以 `<` 为基础的。因此它们判断的结果很可能不同。

下面通过函数子类 `CIStringCompare`，创建了一个不区分大小写的 `set<string>`

```c++
struct CIStringCompare:
    public binary_function<string, string, bool>{ // 该基类见第 40 条
    bool operator()(const string& lhs, const string& rhs) const {
        return ciStringCompare(lhs, rhs); // 见第 35 条的实现
    }
}

set<string, CIStringCompare> ciss;
```

如果插入 "STL" 和 "stl"，则只有第一个字符串会被插入，因为 `insert` 判断相同的标准是等价

```c++
ciss.insert("STL")
ciss.insert("stl") // "stl" 不会被插入
```

成员函数 `ciss.find("stl")` 会返回 `true`。而非成员函数 `find(ciss.begin(), ciss.end(), "stl")` 却会返回 `false`。这也是因为它们对相同的判断标准不一样。

这说明我们应该优先选用成员函数而不是对应的非成员函数（见第 44 条）。因为我们不插入“stl”的原因是“它已经存在于容器中了”，所以判断“stl”是否存在的函数自然要返回真。

而对于非标准的基于散列表的非排序关联容器（如 `hash_map`），则有基于想等和基于等价的两种设计（见第 25 条）。

### 第 20 条：为包含指针的关联容器指定比较类型

下面的函数子类可以作为包含指针的关联容器的比较类型

```c++
strcut DereferenceLess{
    template<typename PtrT> bool operator(PtrT p1, PtrT p2) const {
        return *p1 < *p2;
    }
}
```

### 第 21 条：总是让比较函数在等值时返回 `false`

如果用 `<=` 作为 `set` 的比较函数，`set` 会认为相等的元素并不等价，这是因为`!(x <= x) && !(x <= x)` 的值是 `false`。所以 `set` 可能会存储两个相等的元素，它不再是一个集合。

就算是 `multi_set` 也会出问题。考虑包含两个 10 的 `multi_set<int> s`，它以 `less_equal<int>` 作比较函数。`equal_range(s.begin(), s.end(), 10)` 会返回一个包含“等价”值的区间，而这两个 10 并不等价，因此它们不可能同时出现在这个区间里。

所以，不管关联容器是否允许存储重复的值，都要让比较函数在等值时返回 `false`。

### 第 22 条：不要直接修改 `set` 中的键

`map` 中元素的键是 const 的，而 `set` 并不是，原因是修改 `set` 中元素无关排序的属性是合理的。

一定不要改变 `set` 中元素影响排序的属性，这会破坏容器。

由于标准的模棱两可，某些 `set` 的实现中，所有途径获取的元素都是 const 引用，即不允许修改。这时假设我们一定要修改 `Widget` 类中无关排序的属性，我们可以用强制型别转换来去除 const 性质

```c++
const_cast<Widget&>(*iter).setAttr("123")
```

下面的写法能通过编译，却是错误的

```c++
static_cast<Widget>(*iter).setAttr("123")
```

原因是 `static_cast` 是创建一个临时的匿名对象，它等价于

```c++
((Widget)(*iter).setAttr("123"))
```

可能更简单也更安全的方法是先 `erase` 再 `insert`。

### 第 23 条：考虑用排序的 `vector` 代替关联容器

如果查找几乎从不和插入和删除混在一起，排序的 `vector` 配合 `binary_search`、`lower_bound`、`equal_range` 通常会比 `set` 和 `map` 有更快的速度和更少的内存。

虽然理论时间复杂度都是 `log` 级的，但 `vector` 速度更快的原因主要是它的内存是连续的，这意味着更少的缺页错误。另外二分查找的复杂度常数通常也比平衡二叉树好得多。

对 `set` 的模拟很简单，模拟 `map` 则有一些麻烦。我们在 `vector` 中存储 一个 pair `Data`，在比较函数子类中需要重载三种 `()`

```c++
class Compare {
    public:
    bool operator()(const Data& lhs, const Data& rhs) const;
    bool operator()(const Data::key_type& lhs, const Data& rhs) const;
    bool operator()(const Data& lhs, const Data::key_type& rhs) const;
}
```

这是因为排序函数需要比较两个 `Data` 的大小，而查找函数需要比较 `Data::key_type`和 `Data` 的大小。

### 第 24 条：`map::operator[]` 和 `map::insert`

用 `operator[]` 更新 `map` 中存在的元素通常比 `insert` 更快，而添加新的元素则相反。这在效率至关重要的时候需要做出选择。

### 第 25 条：关注非 STL 的散列容器

## 迭代器

## 算法

### 第 30 条：确保容器容量足够大

用 `insert` 向容器末尾添加新的对象，这样做是错误的

```c++
insert(values.begin(), values.end(), results.end());
```

因为这会尝试为 `*results.end()` 赋值。正确的做法是使用插入迭代器

```c++
insert(values.begin(), values.end(), back_inserter(results));
```

这会自动调用 `push_back`。其他的插入迭代器同理，例如我们可以这样在特定位置插入元素

```c++
insert(values.begin(), values.end(), inserter(results, results.begin()+results.size()/2));
```

当然这样做对连续内存的容器效率并不高。

### 第 31 条：排序有关的选择

找到最小的 20 个数并升序排列在容器的前 20 个位置

```c++
partial_sort(v.begin(), v.end() + 20, v.end());
```

找到最小的 20 个数字并放在前 20 个位置

```c++
nth_element(v.begin(), v.end() + 19, v.end());
```

找到大于 30 的数字放在容器前部

```c++
partition(v.begin(), v.end(), [](int x) { return x > 30; });
```

稳定排序用 `stable_sort` 和 `stable_partition`。更多的基于“能恰好完成功能”，而不是性能来选择算法。

### 第 32 条：在 `remove` 之后 `erase`

`remove` 并不真正删除容器中的元素，因为它只接受两个迭代器作为参数，并不能调用成员函数来删除元素。`remove` 只是把不需要删除的元素放在容器的前部，并返回后一个位置的迭代器。结合 `erase` 才可以真正删除元素

```c++
v.erase(remove(v.begin(), v.end(), 99), v.end());
```

同理，`remove_if` 和 `unique` 也要用类似的方法使用。

而 `list` 的成员函数可以直接达到预期效果，并且更加高效

```c++
l.remove(99);
l.unique();
```

### 第 33 条：小心对包含指针的容器使用 `remove`

这条也是 C++11 之前的历史遗留问题。

### 第 34 条：需要用有序区间作参数的算法

二分查找系列：`binary_search` `lower_bound` `upper_bound` `equal_range`

集合操作：`set_union` `set_intersection` `set_difference` `set_symmetric_difference`

合并：`merge` `inplace_merge`

包含：`includes`

去重：`unique` `unique_copy`

保证算法使用的排序函数和有序区间的排序函数一致

```c++
sort(v.begin(), v.end(), greater<int>());
binary_search(v.begin(), v.end(), 5); // 错误
binary_search(v.begin(), v.end(), 5, greater<int>()); // 正确
```

除 `unique` 和 `unique_copy` 外，所有要求有序区间的函数均使用等价性判断元素相同。这是因为 `unique` 实际上并不要求有序区间，它的标准定义是“删除每一组连续相等的元素，仅保留其中的第一个”，只是通常用来去重时要区间有序才能达到预期效果。

### 第 35 条：实现忽略大小写的字符串比较

```c++
bool ciCharLess(char c1, char c2) {
    return tolower(static_cast<unsigned char>(c1)) < tolower(static_cast<unsigned char>(c2));
}
bool ciStringCompare(const string &s1, const string &s2) {
    return lexicographical_compare(s1.begin(), s1.end(), s2.begin(), s2.end(), ciCharLess);
}
```

`lexicographical_compare` 可以用于任何类型的值区间。

### 第 36 条：实现 `copy_if`

C++11 起 STL 已经提供 `copy_if`

```c++
vector<int> v{1, 2, 3};
vector<int> vv;
vv.resize(v.size());
vv.erase(copy_if(v.begin(), v.end(), vv.begin(), [](int x) { return x > 1; }), vv.end());
```

### 第 37 条：用 `accumulate` 和 `for_each` 进行区间操作

用 `accumulate` 实现自定义统计，需要一个函数子类，它接受当前的统计值和下一个元素，返回一个新的统计值。

统计容器中字符串总长度

```c++
size_t StringLengthSum(size_t sumSoFar, const string &s) {
    return sumSoFar + s.size();
}
auto lengthSum = accumulate(v.begin(), v.end(), static_cast<size_t>(0), StringLengthSum);
```

`accumulate` 的返回值类型和第三个参数相同。

`for_each` 的用法类似，只不过它需要一个 `void` 函数，接受一个元素并执行操作。

## 函数子、函数子类、函数及其他

### 第 38 条：依照按值传递的原则设计函数子类

STL 算法中总是按值传递函数子类，所以我们在设计函数子类时要注意

1. 函数对象尽量小，否则复制开销很大
2. 函数不能是多态的，即不得使用虚函数，否则会产生剥离（见第 3 条）

如果确实需要很大的函数对象或要保留多态性，可以使用 Pimpl Idiom 设计模式，只保存一个指向真正实现类的指针，而实现类可以自由使用虚函数。

### 第 39 条：确保判别式是纯函数

STL 算法很可能会复制传入的判别式函数，因此一定要确保判别式是纯函数。

### 第 40 条：让函数子类可配接

（书中的 `ptr_fun`、`mem_fun`、`mem_fun_ptr`、`bind1st`、`bind2nd`、`not1`、`not2` 都已经过时）

### 第 41 条：函数配接器

（书中的 `ptr_fun`、`mem_fun`、`mem_fun_ptr`、`bind1st`、`bind2nd`、`not1`、`not2` 都已经过时）

**配接器（adapter）** 接受一个函数子类，返回另一个函数子类。C++11 后，我们可以用 `bind` 来作为 STL 算法饿的函数配接器

例如，我们可以这样找到容器中所有大于 40 的数（书中的 `bind_2nd` 已经过时）

```c++
find_if(v.begin(), v.end(), bind(greater<int>(), std::placeholders::_1, 40));
```

用 `for_each` 执行成员函数，成员函数有额外的第一个占位符表示对象本身

```c++
class Widget {
    int x;
    void func(int y) {
        cout << x << " " << y;
    }
};

vector<Widget> v;
// ...
for_each(v.begin(), v.end(), bind(&Widget::func, std::placeholders::_1, 10)); // 输出每个 Widget 的 x 和 10
```

这个方法同样适用于对应的指针（智能指针）容器

```c++
vector<unique_ptr<Widget>> vp;
// ...
for_each(vp.begin(), vp.end(), bind(&Widget::func, std::placeholders::_1, 10));
```

用 `not1` 和 `not2` 可以构造新的函数子，但是对传入的函数有要求，最简单的方法是继承 STL 的类模版

```c++
class LessThan10 : public unary_function<int, bool> {
public:
    bool operator()(const int x) const {
        return x < 10;
    }
};

find_if(v.begin(), v.end(), not1(LessThan10()));
```

C++17 以后，`not_fn` 取代了 `not1` 和 `not2`，且对传入的函数没有要求

### 第 42 条：确保 `less<T>` 和 `operator<` 有相同的含义

当我们需要在 STL 算法中使用不同的比较函数时，不要尝试特化 `less<T>`，而是临时自定义比较函数。
