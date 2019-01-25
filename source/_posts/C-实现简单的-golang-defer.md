---
title: C++ 实现简单的 golang `defer`
date: 2019-01-24 22:42:56
tags: C++
categories: C++
---

最近在看 golang，发现它的 `defer` 关键字很好用，可以将部分代码推迟到函数结束之后执行。

```go
package main

import "fmt"

func main() {
	defer fmt.Println("Defered things")

	fmt.Println("Exit main")
}
```

运行结果为：

```
$ go run test.go
Exit main
Defered things
```

C++ 中暂时还没有类似的功能，但是要自己实现一个类型的也很简单，基本原理就是利用局部对象在函数结束后自动析构的方法。标准库中的 `std::lock_guard` 就使用了这个方法。 

于是可以简单的写出这样一个类：

```C++
class Defer {
public:
    explicit Defer(F f) : _f(f) {}
    ~Defer() {
        _f();
    }
private:
    F _f;
};
```

需要推迟的部分放在`_f`中，对象析构时执行`_f()`。要推迟的东西千奇百怪，不可能为所有东西都定义一个成员，所以简单的，将他们包在一个 `lambda` 表达式里，对象 hold 住一个`lambda` 表达式即可。及 `_f` 保存 `lambda` 表达式。

`lambda` 表达式可以用 `std::function` 来保存，也可以用模板来推导。所以可以这么定义：

```C++
template <typename F>
class Defer {
public:
    explicit Defer(F &&f) : _f(std::forward<F>(f)){}
    ~Defer() {
        _f();
    }


private:
    F _f;
};
```

这个模板类创建对象时，需要指定类型，我们不知道，`decltype` 也不允许推导匿名`lambda` 表达式（lambda-expression in unevaluated context）。可以定义一个帮助函数来创建对象：

```C++
template<typename F>
auto MakeObject(F &&f) {
    // 此时 F 可以被推导出来
    return Defer<F>(std::forward<F>(f));
}
```

于是简单的 `defer` 类就成型了，（禁止拷贝与赋值，以防止创建`lambda`副本，导致`defer`部分被执行多次）：

```C++
#define _CONCAT(a, b) a##b
#define CONCAT(a, b) _CONCAT(a, b)
#define DEFER_OBJ CONCAT(__DEFER_,  CONCAT(__func__, __LINE__))
#define defer(expr) const auto &DEFER_OBJ = MakeObject([&](){ expr })

template <typename F>
class Defer {
public:
    explicit Defer(F &&f) : _f(std::forward<F>(f)) {}
    ~Defer() {
        _f();
    }

    Defer(Defer &&that) : _f(std::move(that._f)) {}

    Defer() = delete;
    Defer(const Defer &) = delete;
    void operator=(const Defer &) = delete;
    void operator=(Defer &&) = delete;

private:
    F _f;
};

template<typename F>
auto MakeObject(F &&f) {
    return Defer<F>(std::forward<F>(f));
}
```

简单的验证一下：

```C++
void func() {
    defer( std::cout << "deferred something 1 in function" << std::endl; );
    defer( std::cout << "deferred something 2 in function" << std::endl; );
    std::cout << "entering function" << std::endl;
    std::cout << "exiting function" << std::endl;
}

int main() {
    defer( std::cout << "deferred something... 1" << std::endl; );
    defer( std::cout << "deferred something... 2" << std::endl; );

    defer( func(); );

    std::cout << "bye..." << std::endl;
}
```

输出：

```
bye...
entering function
exiting function
deferred something 2 in function
deferred something 1 in function
deferred something... 2
deferred something... 1
```

除了执行顺序倒序之外（析构与构造反序），看起来似乎还可以。

[完整代码](https://github.com/anonymouss/my-exercises/tree/master/cpp/defer)