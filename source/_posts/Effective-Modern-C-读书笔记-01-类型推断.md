---
title: 'Effective Modern C++ 读书笔记: 01 类型推断'
date: 2019-01-22 14:32:25
tags:
    - C++
    - Effective Modern C++
    - Modern C++
    - C++ 11
    - C++ 14
categories: C++
---

## Chapter 01. 类型推导

* 条目 1：理解模板类型推导
* 条目 2：理解`auto`类型推导
* 条目 3：理解`decltype`
* 条目 4：知道如何查看编译器推导出的类型

### 0. 准备

为了可以直观的知道编译器推导出的类型结果是否与我们所想的一致，首先来定义一个可以正确完整显示变量类型的帮助宏。由于C++标准库中的[`typeid(expr).name()`](https://zh.cppreference.com/w/cpp/language/typeid)会忽略`cv`限定符，并且其结果在gcc下不够直观（诸如：*`NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE`*，其实是`std::string`）。因此，这里直接使用**条目4**中提到的**`boost`**库中提供的`boost::typeindex::typeid_with_cvr`的方法。*PS: `typeid`可以参考此文章自定义模板来获取完整类型名：[怎样在C++中获得完整的类型名称](https://www.cnblogs.com/zfyouxi/p/5060288.html)*

宏定义如下：

```C++
#include <boost/type_index.hpp>

#define ECHO_TYPE(T, U)                                             \
{                                                                   \
    using boost::typeindex::type_id_with_cvr;                       \
    std::cout.width(10), std::cout.setf(std::ios::left);            \
    std::cout << #U << " = "                                        \
              << type_id_with_cvr<T>().pretty_name()                \
              << std::endl;                                         \
}                                                                   \

#define ECHO_T_TYPE(T)          ECHO_TYPE(T, T)
#define ECHO_PARAM_TYPE(param)  ECHO_TYPE(decltype(param), param)
#define ECHO()                                                      \
{                                                                   \
    std::cout << std::string(30, '-') << std::endl;                 \
    ECHO_T_TYPE(T);                                                 \
    ECHO_PARAM_TYPE(param);                                         \
}                                                                   \
```

### 1. 模板推导

假定模板与调用函数定义如下：

```C++
template <typename T>
void f(paramType param);

f(expr);
```

`T`和`paramType`由编译器在编译期间根据`expr`推导得到，`T`的推导结果和`paramType`的声明形式有关。模板推导有三种情况：

#### 1.1 `paramType`声明为引用或指针（非通用引用`T&&` / universal reference）

`T`的推导规则：

- 如果`expr`为引用，则忽略引用部分

- 然后匹配`expr`的类型和`paramType`的声明类型

```C++
// 这种情形的模板申明如下
template <typename T>
void f1(T &param) {
    ECHO();
}
// 调用函数
int v = 10;              // int
int &rv = v;             // int&
const int cv = v;        // const int
const int &crv = v;      // const int&
f1(v);                   // T is int, paramType is int&
f1(rv);                  // T is int, paramType is int&
f1(cv);                  // T is const int, paramType is const int&
f1(crv);                 // T is const int, paramType is const int&
// 实际运行结果
/*
------------------------------
T          = int
param      = int&
------------------------------
T          = int
param      = int&
------------------------------
T          = int const
param      = int const&
------------------------------
T          = int const
param      = int const&
*/
```

注意，**`T`的类型推导受到`paramType`声明形式的影响**。如果这里`paramType`声明为`const T&`形式，此时如果`expr`是`const int&`类型，根据推导规则，首先去除引用剩下`const int`，但由于此时`paramType`已经声明为`const`，于是`T`不在需要`const`，最终**`T`的推导结果为`int`，`paramType`为`const int&`**。另外，这种情况下，`paramType`声明为常量左值引用形式，如果是非常量左值引用形式，即`T&`，那就无法推导类似于`f(2)`这种情况（无法调用，编译错误）。因为`2`是纯右值(prvaue)，对于左值引用，除非它是`const`的，否则不可以指向右值。

#### 1.2 `paramType`声明为通用引用`T&&`

`T`的推导规则：

- 若`expr`为左值，**`T`和`paramType`都会被推导为左值引用类型（`T&`）**。这是很特殊的一点！一方面，只有这种情况时，`T`才会被推导为引用类型；另一方面，即便此时`paramType`被声明为右值引用的形式，其推导结果仍然为左值引用。

- 若`expr`为右值，则规则同`1.1`。

```C++
// 模板声明
template <typename T>
void f2(T &&param) {
    ECHO();
}
// 调用函数
f2(v);                  // T and paramType are both int&
f2(rv);                 // T and paramType are both int&
f2(cv);                 // T and paramType are both const int&
f2(crv);                // T and paramType are both const int&
f2(std::move(v));       // T is int, paramType is int&&
f2(std::move(rv));      // T is int, paramType is int&&
f2(std::move(cv));      // T is const int, paramType is int&&
f2(std::move(crv));     // T is const int, paramType is int&&
// 运行结果
/*
------------------------------
T          = int&
param      = int&
------------------------------
T          = int&
param      = int&
------------------------------
T          = int const&
param      = int const&
------------------------------
T          = int const&
param      = int const&
------------------------------
T          = int
param      = int&&
------------------------------
T          = int
param      = int&&
------------------------------
T          = int const
param      = int const&&
------------------------------
T          = int const
param      = int const&&
*/
```

#### 1.3 `paramType`既不是引用也不指针形式（值传递）

`T`的推导规则：

- 和前面一样，若`expr`是引用，则忽略引用部分。

- 忽略引用之后，若`expr`是`const`或`volatile`的，`const`和`volatile`也忽略。即**引用、`const`、`volatile`**在这种情况下通通忽略。这是合理的，因为值传递是实参的一份临时拷贝，对其修改并不会影响到实参，所以不必受到实参cv条件限制。

```C++
// 模板定义
template <typename T>
void f3(T param) {
    ECHO();
}
// 函数调用
f3(cv);                 // both T and paramType are int
f3(crv);                // both T and paramType are int
const char *str1{"something"};  // str1 is const char*
// !note! ISO C++ forbids `const char*` to `char *`
char *const str2{"something"};  // str2 is char* const
f3(str1);               // both T and paramType is const char*
f3(str2);               // both T and paramType is char*
// 输出结果
/*
------------------------------
T          = int
param      = int
------------------------------
T          = int
param      = int
------------------------------
T          = char const*
param      = char const*
------------------------------
T          = char*
param      = char*
*/
```

#### *1.4 数组实参

数组通常和指针联系在一起，因而数组形参通常都可以声明为指针形式。如`void func(int *arr);`和`void func(int arr[]);`。这两种形参声明，数组作实参传递时都会退化为指针，因此其推导规则和指针是一样的。

但是，**当数组形参声明为引用时**，数组就不会退化为指针了（包括通用引用）。这时的推导规则就应当使用数组本身类型来推导。

```C++
int a[] = {1, 2, 3};    // int[3]

f1(a);                  // T is int[3], paramType is int(&)[3]
f2(a);                  // both T is int(&)[3]
f3(a);                  // both T and paramType are int*
```

这样特性还可以用来编译期获取数组维度：

```C++
template <typename T, std::size_t N>
constexpr std::size_t getArrayLength(const T (&)[N]) {
    return N;
}
auto len = getArrayLength(a);
// 编译期获得数组长度还有另外两种方法
auto len = sizeof(a) / sizeof(a[0]);            // 使用sizeof
auto len = std::extent<decltype(a)>::value;     // 使用std::extent
```

#### *1.5 函数实参

函数实参和数组实参一样，也会退化为指针，推导规则和数组类似。

```C++
void funcToDeduce(int);             // type: void(int)

f1(funcToDeduce);                   // T is void(int), paramType is void(&)(int)
f2(funcToDeduce);                   // both T and paramType is void(&)(int)
f3(funcToDeduce);                   // both T and paramType is void(*)(int）
```

### 2. `auto`推导

`auto`推导和模板推导的规则基本一致，`auto`就相当于模板推导中的`T`。`auto`推导完全可以套用模板推导规则，但是这里有一个特例。即从初始化列表推导时要特殊注意，模板是不能从初始化列表推导的。

```C++
auto x1 = 1;            // int
auto x2(1);             // int
auto x3 = {1};          // std::initializer_list<int>
auto x4{1};             // int since C++17, std::initializer_list<int> before C++17
auto x5 = {1, 2};       // obviously std::initializer_list<int>
// auto x6{1, 2};       // FORBIDDEN
ECHO_PARAM_TYPE(x1);
ECHO_PARAM_TYPE(x2);
ECHO_PARAM_TYPE(x3);
ECHO_PARAM_TYPE(x4);
ECHO_PARAM_TYPE(x5);
// 输出
/*
x1         = int
x2         = int
x3         = std::initializer_list<int>
x4         = int
x5         = std::initializer_list<int>
*/
```

**NOTE**: *关于`x4`，提案*[N3932](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3922.html)*修改了`auto`的推导规则，所以从C++17起，`x4`被推导为`int`。但是在我的机器上（GCC 8.2），即使限制使用C++14，`x4`也被推导为`int`而不是`std::initializer_list<int>`。并且，*[CppReference](https://zh.cppreference.com/w/cpp/language/auto)*中说，`x6`在C++17起视为错误，之前会被推导为`std::initializer_list<int>`，而我的编译器上即使限制为C++14，仍然会被视为错误。*

需要传递初始化列表实参给模板推导时，模板`paramType`需要显式的声明为初始化列表形式，即：

```C++
template <typename T>
void f4(std::initializer_list<T> &&param) {
    ECHO();
}

f4({1});                // T is int, paramType is std::initializer_list<int>&&
// 输出
/*
------------------------------
T          = int
param      = std::initializer_list<int>&&
*/
```

此外，从C++14起，`auto`可以推导函数返回类型（C++11需要显示指定或者使用尾置返回类型），并且可以修饰`lambda`表达式形参。作为这些用法是，`auto`使用模板的推导规则，因而无法推导列表初始化

```C++
// auto generateInitList() {
//     return {1};              // Compile Error, decltype(auto) 也不行
// }

// std::vector<int> vec;
// auto resetVector = [&] (const auto& v) {
//     vec = v;
// };
// resetVector({1});            // 编译错误
```

### 3. `decltype`推导

`decltype`与`auto`和模板不同，它将返回精确的类型（`auto`和模板在某些情况会去cv限定符和引用），`ECHO_PARAM_TYPE`宏就是使用`decltype`来获取对象精确类型的，

有时候，模板函数的返回值需要根据参数来推导，C++11中可以使用`decltype`尾置返回类型（如前所述，C++14起才支持`auto`直接推导函数返回类型），

```C++
template <typename Container, typename Index>
auto getIndexAt1(Container& c, Index i) -> decltype(c[i]) {
    return c[i];
}
```

假设传入的容器为`std::vector<int>`，那么返回类型为`int&`。

C++14起，`auto`可以直接从`return`语句推导函数返回值了，即不在需要尾置返回类型。这种写法方便多了，但是也存在一些陷阱。因为这里`auto`使用的是模板推导规则，它可能丢弃cv限定符与引用信息，不像`decltype`可以推导精确类型。比如：

```C++
template <typename Container, typename Index>
auto getIndexAt2(Container& c, Index i) {
    return c[i];
}
```

假设传入的容器为`std::vector<int>`，则推导结果为`int`。和前一个尾置返回类型的例子相比，它缺失了引用信息。当然我们可以使用`auto &`来补上，但这并不make sense，毕竟这是模板函数，下次类型就是非引用怎么办？解决办法是使用**`decltype(auto)`**（C++14）

```C++
template <typename Container, typename Index>
decltype(auto) getIndexAt3(Container& c, Index i) {
    return c[i];
}
```

这样就没有问题了，完美！

还有更加完善的做法，就是使用完美转发（perfect forwarding）

```C++
// perfect forwarding
template <typename Container, typename Index>
decltype(auto) getIndexAt4(Container &&c, Index i) {
    return std::forward<Container>(c)[i];
}
```

完美转发后面会有专题去讲，就此带过。

`decltype(e)`的[推导规则](https://zh.cppreference.com/w/cpp/language/decltype)：

1. 若`e`是没有使用小括号`()`括起来的表达式，则其推导结果就是其声明的类型。

2. 若`e`是`T`类型的`xvalue`，则推导结果为`T&&`，右值引用。

3. 若`e`是`T`类型的`lvalue`，则推导结果为`T&`，左值引用。（**若同时满足 1，优先使用 1的规则。所以，需要使用这条规则通常要加括号括起来**）。

4. 若`e`是`T`类型的`prvalue`，则推导结果为`T`。（字面值常量，临时变量等）。

**NOTE**：`int a = 1, b = 2;` 此时`decltype((a + b))`是`int`而不是`int&`。因为虽然加了括号，`a + b`是值为`prvalue`。

[**related codes**](https://gitlab.com/jcong/effective-modern-cpp-notes/blob/master/codes/01-deducing_types.cpp)