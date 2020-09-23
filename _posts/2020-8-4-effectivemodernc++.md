---
# layout: post
title: Effective Modern C++ Reading Notes
categories: 
    - Reading Notes
tag:
    - C++
---

*Effective Modern C++* reading notes

## Type Dedution

### `auto` and `decltype(auto)`

`auto` follows the template argument deduction rules and is always an object type; `decltype(auto)` follows the `decltype` rules for deducing reference types based on value categories. So if we have

```c++
int x;
int && f();
```

then

    expression    auto       decltype(auto)
    ----------------------------------------
    10            int        int
    x             int        int
    (x)           int        int &
    f()           int        int &&


## Rvalue References, Move Semantics, and Perfect Forwarding

### lvalue and rvalue

C++11’s most pervasive feature is probably move semantics, and the foundation of move semantics is distinguishing expressions that are rvalues from those that are lvalues.

A useful heuristic to determine whether an expression is an lvalue is to ask if you can take its address. If you can, it typically is. If you can’t, it’s usually an rvalue. 

It’s especially important to remember this when dealing with a parameter of rvalue reference type, because the parameter itself is always an lvalue:

```c++
class Widget {
public:
  Widget(Widget&& rhs); // rhs is an lvalue, though it has an rvalue reference type 
};
```
**Remember the type of an expression is independent of whether the expression is an lvalue or an rvalue.**

### `std::move` and `std::forward`

At runtime, neither of `std::move` and `std::forward` does anything at all.

Actually, `std::move` **unconditionally** casts its argument to an rvalue, while `std::forward` performs this cast only if a particular condition is fulfilled.

Here's a sample implementation of `std::move` in C++11

```c++
template<typename T> // in namespace std
typename remove_reference<T>::type&&
move(T&& param)
{
  using ReturnType = typename remove_reference<T>::type&&;
  return static_cast<ReturnType>(param);
}
```

In C++14, thanks to **function return type dedution** and STL's alias template `std::remove_reference_t`, `std::move` can be written this way

```c++
template<typename T> // C++14; still in namespace std
decltype(auto) move(T&& param)
{
  using ReturnType = remove_reference_t<T>&&;
  return static_cast<ReturnType>(param);
}
```
 