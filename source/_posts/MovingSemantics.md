---
date: '2024-03-13T20:38:20+08:00'
draft: false
tags: ["Programming", "C++"]
ShowToc: true
title: 'std::move'
---

std::move 是移动语义中常见的一种标准库函数，它的作用是将一个左值转发为右值。如果读者了解完美转发（std::forward）的话，就会发现 std::move 其实是完美转发的一个子集。在一点在我之前的[博客](../forwarding/)中也提到过。在 [cppreference.com](https://en.cppreference.com/w/cpp/utility/move)，对 std::move 下了如下的定义：

> `std::move` is used to *indicate* that an object t may be "moved from", i.e. allowing the efficient transfer of resources from t to another object. In particular, `std::move` produces an [xvalue expression](https://en.cppreference.com/w/cpp/language/value_category) that identifies its argument t. It is exactly equivalent to a `static_cast` to an rvalue reference type.

std::move 的声明如下：

```cpp
template< class T >
typename std::remove_reference<T>::type&& move( T&& t ) noexcept;
```

std::move 的声明说明了它的输出的类型是 T&&，也就是 T 的右值引用。所以当 std::move 的结果被当作右值使用的时候，会激活右值引用形参的重载函数。我们看个例子：

```cpp
void foo(int&) { std::cout << "lvalue overload" << std::endl; }
void foo(int&&) { std::cout << "rvalue overload" << std::endl; }

int main() {
  int x = 1;
  foo(x); // lvalue_overload
  foo(std::move(x)); // rvalue overload
}
```

任何人都会好奇，如果我不这样使用 std::move 的结果呢？让我们想一想，如果我们用一个局部变量将 std::move 的结果保存下来，那它将是一个右值引用类型的左值，当它被传递到 foo 中时，只会激活左值引用类型形参的重载函数（根据引用折叠，T&& &->T&）。我们在 [compiler explorer](https://godbolt.org/z/z1h36bMcs) 中验证一下。

读者应该还会好奇，能不能传一个右值给 std::move 呢？看代码应该是可以的，因为 std::move 的形参是一个万能引用。不过还是让我们[验证一下](https://godbolt.org/z/vWoYvedn5)。

确实可以！我们直接把 std::move 的结果传给了 std::move，激活了 foo 的右值重载版本。不过这么做有什么好处呢？为什么需要把右值转成右值呢？这是一个开放的问题，也许在不清楚入参会是左值还是右值的时候会派上用场吧。

还有一个问题，如果显示地实例化 std::move 的模板参数会发生什么？需要注意的是，在这种情况下，因为不涉及到模板参数的推导，所以 std::move 的万能引用退化成了简单的右值引用。结合引用折叠的规则，可以有以下这些结论：

```cpp
int x;
std::move<int>(x); // wrong, int&&，cannot bind lvalue to rvalue reference
std::move<int>(std::move(x)); // ok, int&&
std::move<int&>(x); // ok, int& && -> int&
std::move<int&>(std::move(x)); // wrong, int&, cannot bind rvalue to non-const lvalue reference
std::move<int&&>(x); // wrong, int&& && -> int&&
std::move<int&&>(std::move(x)); // ok, int&&
```

对引用折叠，[完美转发](../forwarding/)和[万能引用](../UniversalReference/)不太了解的读者可以参考我的另外几篇博客，那里讲得比较详细。
