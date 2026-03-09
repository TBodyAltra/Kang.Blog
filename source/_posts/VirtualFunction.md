---
date: '2024-03-17T11:46:51+08:00'
draft: false
tags: ["Programming", "C++"]
ShowToc: true
title: '虚函数 (Virtual Function)'
---

# 简介

虚函数是 C++ 中的一种成员函数，它的开头用关键字 “virtual” 进行修饰。虚函数是用于实现运行时多态的一种方式。

# 虚函数的简单例子

```cpp
class Base {
public:
    virtual void foo();
};

class Derived : public Base {
public:
    void foo();
}
```

在以上例子中，Base 声明了一个虚函数 foo，它的派生类 Derived，重载了这个 foo 函数。在运行时，如果指针或者引用指向的将是 Base，那么调用的将是 Base::foo；如指针或者引用指向的是 Derived 对象，那么调用的将是 Derived::foo。

# 运行时多态

```cpp
class Base {
public:
    virtual void foo() { std::cout << "Base foo" << std::endl; };
};

class Derived : public Base {
public:
    void foo() { std::cout << "Derived foo" << std::endl; };
};

int main() {
  Derived d;
  Base *b1 = &d;
  Base &b2 = d;
  b1->foo();
  b2.foo();
  return 0;
}

Output:
Derived foo
Derived foo
```

在上面的例子中，我们将 Base 类型的指针和引用指向 Derived 对象，在调用 foo 的时候，实际调用的是 Derived::foo，这就是运行时多态。当有函数的接口只接受基类对象的指针类型的时候，我们可以传入实际指向子类对象的指针，并且可以成功调用子类对象的成员函数。

# 虚函数表

现代编译器是如何实现基于虚函数的运行时多态的呢？C++标准并没有给出具体的规范，不过大多数的编译器都是通过虚函数表来实现的。具体地说，如果基类有虚函数的话，那么编译器会给基类和派生类都生成一个指针成员变量，这个指针指向的是虚函数表。比方说 Base 类的虚函数表中会有一个记录，将 void foo() 这个函数指向 Base::foo 的函数地址；而在 Derived 类的虚函数表中会有一个记录，将 void foo() 这个函数指向 Derived::foo 的函数地址。因此，在实际调用的时候，根据实际的对象，将会得到实际的虚函数表，也就是得到实际的虚函数地址。

## 虚函数表增加类的内存占用

通过计算类的大小可以让我们一窥虚函数表的存在：

```cpp
#include <iostream>

class Base1 {
public:
  virtual void foo();
};

class Derived1 : public Base1 {
public:
  void foo();
};

class Base2 {
public:
  int data;
};

class Derived2 : public Base2 {};

int main() {
  std::cout << "Size of void*: " << sizeof(void*) << std::endl;
  std::cout << "Size of Base1: " << sizeof(Base1) << std::endl;
  std::cout << "Size of Derived1: " << sizeof(Derived1) << std::endl;
  std::cout << "Size of Base2: " << sizeof(Base2) << std::endl;
  std::cout << "Size of Derived2: " << sizeof(Derived2) << std::endl;
  return 0;
}

Output:
Size of void*: 8
Size of Base1: 8
Size of Derived1: 8
Size of Base2: 4
Size of Derived2: 4
```

在上面的例子中，Base1 和 Derived1 因为有虚函数表指针的存在，所以它们的大小是 8 byte；作为对比，Base2 和 Derived2 只有 int 类型的成员变量，因此它们的大小是 4 byte。

那么，如果 Derived1 不对 foo 进行重载，它还会有虚函数表指针吗？答案是肯定的，让我们看下面的例子：

```cpp
#include <iostream>

class Base {
public:
  virtual void foo() { std::cout << "Base foo" << std::endl; };
};

class Derived : public Base {};

int main() {
  std::cout << "Size of Base: " << sizeof(Base) << std::endl;
  std::cout << "Size of Derived: " << sizeof(Derived) << std::endl;
  Derived d;
  Base *b = &d;
  b->foo();
  return 0;
}

Output:
Size of Base: 8
Size of Derived: 8
Base foo
```

在上面的例子我们可以看到，即使没有重载虚函数 foo，Derived 的大小依然是 8 byte，说明它依然有虚函数表指针。只不过，现在 Drived 的虚函数表中指向的是唯一存在的 Base::foo。

# 虚析构函数

当基类有虚函数的时候，需要同时确保基类有虚析构函数，不然可能会出现未定义的结果。这是为什么呢？让我们看下面的例子：

```cpp
#include <iostream>

class Base {
public:
  virtual void foo() {};
  ~Base() { std::cout << "Base destructor" << std::endl; }
};

class Derived : public Base {
public:
  ~Derived() { std::cout << "Derived destructor" << std::endl; }
};

int main() {
  Derived *d = new Derived();
  Base *b = d;
  delete b;
  return 0;
}

Output:
Base destructor
```

你看，如果我们把基类的指针传给 delete，delete 是不知道入参这个指针到底指向的是哪个对象的，所以 delete 只能根据入参指针的类型来调用这个类型对应的析构函数，在这个例子中，就是调用了 Base::~Base()。但是这样就有问题了，因为我们实际上创建的对象是一个 Derived 的对象，而 Derived::~Derived() 没有被调用，所以可能会出现内存泄漏或者其他未定义的问题。

为了避免这个问题，我们需要将基类的析构函数声明为虚函数，这样 delete 就可以通过入参指针的虚函数表来找到实际应该调用的析构函数。如下所示：

```cpp
#include <iostream>

class Base {
public:
  virtual void foo() {};
  virtual ~Base() { std::cout << "Base destructor" << std::endl; }
};

class Derived : public Base {
public:
  ~Derived() { std::cout << "Derived destructor" << std::endl; }
};

int main() {
  Derived *d = new Derived();
  Base *b = d;
  delete b;
  return 0;
}

Output:
Derived destructor
Base destructor
```

## 没有虚析构函数也不致命的场景

可不可以不使用虚析构函数呢？比方说以下例子：

```cpp
int main() {
  Derived *d = new Derived();
  Base *b = d;
  delete d; // pass d to delete
  return 0;
}

Output:
Derived destructor
Base destructor
```

或者：

```cpp
int main() {
  Derived d; // do NOT allocate on heap, so no delete is called
  Base *b = &d;
  return 0;
}

Output:
Derived destructor
Base destructor
```

不过，作为一个好习惯，当基类有虚函数的时候，最好还是声明虚析构函数，这可以减少你很多的 debug 的时间。

# 未完待续

后续我们将通过 gdb/readelf/objdump 看一下虚函数表在内存中的表示，以及一种特别的虚函数：纯虚函数。

# References

[Demystifying Virtual Tables in C++ - Part 3 Virtual Tables](https://martinkysel.com/demystifying-virtual-tables-in-c-part-3-virtual-tables/)

[C++: Deleting destructors and virtual operator delete](https://www.reddit.com/r/cpp/comments/3huvd1/c_deleting_destructors_and_virtual_operator_delete/)

[C++: Deleting destructors and virtual operator delete - Eli Bendersky's website](https://eli.thegreenplace.net/2015/c-deleting-destructors-and-virtual-operator-delete/)
