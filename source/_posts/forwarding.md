---
date: '2024-03-13T20:34:20+08:00'
draft: false
tags: ["Programming", "C++"]
ShowToc: true
title: '完美转发 (std::forward)'
---

完美转发（std::forward）是一个标准库函数。它的作用是什么呢？简而言之，它可以把左值当作左值或者右值转发出去，也可以把右值当作右值转发出去。这段像绕口令一样的描述不是我的原创，而是翻译自 cppreference.com：

> 1) Forwards lvalues as either lvalues or as rvalues, depending on T.  
> 2) Forwards rvalues as rvalues and prohibits forwarding of rvalues as lvalues.  

其目的，读者诸君应该也能大概猜到，凡是涉及到左值右值的，大多是为了实现移动语义。

# 逐条解释

我们先看一下标准库中对 std::forward 的声明。结合一下这段代码，我们可以尝试理解一下上面的那段绕口令。

```cpp
template <class T> T&& forward(typename remove_reference<T>::type& arg) noexcept;
template <class T> T&& forward(typename remove_reference<T>::type&& arg) noexcept;
```

### 1. 将左值转发为左值

```cpp
#include <iostream>
#include <utility>

using namespace std;

struct Foo {
    Foo() { std::cout << "Foo constructor." << std::endl; }
};

struct Bar {
    Bar(Foo&) { std::cout << "Bar copy constructor." << std::endl; }
    Bar(Foo&&) { std::cout << "Bar move constructor." << std::endl; }
};

int main() {
    Foo foo;
    auto b1 = Bar(std::forward<Foo&>(foo));
    auto b2 = Bar(foo);
}

Output:
Foo constructor.
Bar copy constructor.
Bar copy constructor.
```

注意我们调用 std::forward 的时候，实例化的模板参数是 Foo&，因为入参 foo 是左值，所以实例化的是 std::forward 的第一种声明。用 Foo& 替换 T 之后我们可以看到，std::forward 的返回类型为 Foo& &&，而根据引用折叠规则，这一类型实际为 Foo&，因此返回类型为 Foo&，也就是 Foo 的左值引用。所以，Bar 的构造函数看到的入参是一个 Foo& 类型的右值（我们直接把 std::forward 的结果传递给了 Bar 的构造函数），因此激活的是拥有 Foo& 形参的构造函数，也就是 “copy constructor”。可以看到这一调用的结果跟我们直接传递一个左值的 Foo 对象是一样的。因此，我们称在这种情况下，将左值转发为了左值。

### 2. 将左值转发为右值

还是考虑以上的例子，不过我们稍微改动一下调用 std::forward 的方式。

```cpp
#include <iostream>
#include <utility>

using namespace std;

struct Foo {
    Foo() { std::cout << "Foo constructor." << std::endl; }
};

struct Bar {
    Bar(Foo&) { std::cout << "Bar copy constructor." << std::endl; }
    Bar(Foo&&) { std::cout << "Bar move constructor." << std::endl; }
};

int main() {
    Foo foo;
    auto b1 = Bar(std::forward<Foo>(foo));
    auto b2 = Bar(Foo());
}

Output:
Foo constructor.
Bar move constructor.
Foo constructor.
Bar move constructor.
```

细心的读者可能发现了，我们这次没有用 Foo& 来实例化 std::forward 的模板参数，而是用了 Foo。结果有什么不同呢？还是按照上述的推理流程，我们用 Foo 替换 T，发现 std::forward 的第一种声明的返回类型变成了 Foo&&，也就是说，返回类型是 Foo 的右值引用。那么，此时我们传递给 Bar 的构造函数的入参就变成了 Foo&& 类型的右值，所以对应的构造函数应该是 ”move constructor“ （Foo&& && 折叠为 Foo&&）。我们发现这种调用方式的结果跟直接传递一个 Foo 的右值是一样的，因此就称这种情况下我们把一个左值转发成了右值。

> C++ 中有一个独立的库函数 std::move，用于实现将左值转发为右值。它的实现其实就是 std::forward 的一个子集。  

### 3. 将右值转发为右值

```cpp
#include <iostream>
#include <utility>

using namespace std;

struct Foo {
    Foo() { std::cout << "Foo constructor." << std::endl; }
};

struct Bar {
    Bar(Foo&) { std::cout << "Bar copy constructor." << std::endl; }
    Bar(Foo&&) { std::cout << "Bar move constructor." << std::endl; }
};

int main() {
    auto b1 = Bar(std::forward<Foo>(Foo()));
    auto b2 = Bar(Foo());
}

Output:
Foo constructor.
Bar move constructor.
Foo constructor.
Bar move constructor.
```

这次我们传递了一个 Foo 类型的右值给 std::forward。显然，这次调用的是 std::forward 的第二种声明，结果的类型是 Foo&&。因此我们传递了一个 Foo&& 类型的右值给 Bar 的构造函数，激活了 ”move constructor“。这种调用方式的结果跟直接传递一个 Foo 类型的右值给 Bar 的构造函数时一样的，因此也就是将右值转发为了右值。

### ~~4. 将右值转发为左值（不可能）~~

这是行不通的，如果读者比较好奇的话，可以看下面的例子，我在 [compiler explorer](https://godbolt.org/z/5adEr664e) 上进行了验证。推导过程也很简单，有兴趣的读者朋友可以自行研究一下。

# 完美转发有啥用

完美转发这么神奇，那么它到底有什么用呢？在我们上面的例子中，好像都可以用常规的手段来达到 std::forward 的目的，为什么需要 std::forward 呢？

简而言之，在我们定义函数的时候有用。上面的例子中，都是通过手动的方式构造 Bar 的构造函数的入参，因此看不出来 std::forward 的不可或缺的优势。但如果我们定义了一个函数，需要自动实现转发为左值或者右值的功能的时候，怎么办呢？这就需要请一个外援了，那就是[万能引用（universal reference）](../UniversalReference/)。考虑以下代码：

```cpp
#include <iostream>
#include <utility>

using namespace std;

struct Foo {
    Foo() { std::cout << "Foo constructor." << std::endl; }
};

struct Bar {
    Bar(Foo&) { std::cout << "Bar copy constructor." << std::endl; }
    Bar(Foo&&) { std::cout << "Bar move constructor." << std::endl; }
};

template <typename T>
void wrapper1(T&& foo) {
    auto b1 = Bar(std::forward<T>(foo));
    // ...
}

template <typename T>
void wrapper2(T&& foo) {
    auto b2 = Bar(foo);
    // ...
}

int main() {
    Foo foo;
    std::cout << "wrapper1:" << std::endl;
    wrapper1(foo);
    wrapper1(Foo());
    std::cout << "wrapper2:" << std::endl;
    wrapper2(foo);
    wrapper2(Foo());
}

Output:
Foo constructor.
wrapper1:
Bar copy constructor.
Foo constructor.
Bar move constructor.
wrapper2:
Bar copy constructor.
Foo constructor.
Bar copy constructor.
```

在以上代码中，有两个函数，分别是 wrapper1 和 wrapper2，它们各自构造了一个 Bar 的对象，然后接着做自己的事情。当入参是左值时，wrapper1 调用的是 Bar 的拷贝构造函数；当入参是右值时，wrapper1 调用的是 Bar 的移动构造函数。而对于 wrapper2，不管入参是左值还是右值，它构造 Bar 时调用的总是拷贝构造函数。我们知道移动构造函数总是优于拷贝构造函数，因此对于 wrapper1 能够将右值的语义传递下去，是一个很好的功能。这就是  std::forward 带来的好处。

如果读者对于万能引用比较熟悉的话，应该不难理解上面这个例子的工作原理，这里就不作赘述了。不太清楚的读者可以参考我的另一篇[博文](../UniversalReference/)。

# 必须得是 std::forward<T> 的形式

在上面的例子中，完美转发都是通过 std::forward<T> 这种方式来调用的，也就是直接显式地把模板参数实例化，这也是最常见到的调用方式。那么可不可以不这样传递 T，而是让模板去根据我们的入参去进行型别推导呢？很遗憾，不可以。

```cpp
std::forward<Foo>(foo);
std::forward(foo); // false
```

回顾以下 std::forward 的声明：

```cpp
template <class T> T&& forward(typename remove_reference<T>::type& arg) noexcept;
template <class T> T&& forward(typename remove_reference<T>::type&& arg) noexcept;
```

形参 arg 的类型跟 T 并不是一一对应的，比方说如果入参是 Foo& 类型的左值，那么考虑到 remove_reference 的存在，T 可以是 Foo，Foo&，Foo&& 中的任意一种，因此无法推导得到 T 的实际类型。所以 T 的类型必须要手动实例化。
