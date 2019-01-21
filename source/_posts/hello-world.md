---
title: Hello World
tags:
    - site_test # add tag
categories: site_test # add category
comments: true # enable comments
mathjax: true # REQUIRED!!!
---

# Sample post

## basics

- item 1 
- item 2
    - sub-item

*some italic chars* some normal chars **some bold chars** ~~strikethrough~~

[hyperlink](/)

## table

| Type | Size   | Type | Size   |
| ---  | ---    | ---  | ---    |
| int  | 4 bytes| long | 8 bytes|
| short| 2 bytes| byte | 1 byte |

## codes

inline code `func() { fmt.Println("Hello, World!) }()`

code block

```C++
#include <iostream>

int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}
```

## math

> may not work now

inline equation $ E = m \times c^2 $

$$ f(z) = \frac{1}{1 + \exp^{-z}} $$

## reference

- Hexo: https://hexo.io/zh-cn/docs/configuration
- Next: http://theme-next.iissnan.com/getting-started.html