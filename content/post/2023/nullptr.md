---
title: "nullptr 是指针类型吗？如何用C++的方式把\"T*\"转换成\"void*\" |
Is nullptr of Pointer Type? How to Convert \"T*\" to \"void*\" by C++ Way"
date: 2023-06-15T19:58:20+08:00
lastmod: 2023-06-15T19:58:20+08:00
draft: false
tags: ["C++"]
categories: ["技术"]
#toc: false
#math: false
---

## nullptr是指针类型吗？
`nullptr`是C++里预定义的一个变量，它的类型是`std::nullptr_t`。
判断一个类型是否是指针类型，可以用`std::is_pointer`来判断。
测试`std::nullptr_t`是否是指针类型的代码如下：

```C++
#include <myostream.h>
myostream::ostream mycout(std::cout.rdbuf());
#define watch(...) MYOSTREAM_WATCH(mycout, " = ", "\n", "\n\n", __VA_ARGS__)

int main() {
    // 确认nullptr就是std::nullptr_t类型
    static_assert(std::is_same_v<decltype(nullptr), std::nullptr_t>, "Never happpen");

    watch(std::is_pointer_v<std::nullptr_t>,
          std::is_member_pointer_v<std::nullptr_t>,
          std::is_pointer_v<std::nullptr_t *>
    );
    return 0;
}
```

输出：

```Txt
std::is_pointer_v<std::nullptr_t> = 0
std::is_member_pointer_v<std::nullptr_t> = 0
std::is_pointer_v<std::nullptr_t *> = 1
```

可见`std::nullptr_t`并不是一个指针类型。
只不过我们平时常用`nullptr`来赋值给任意指针类型，会给人一种`std::nullptr_t`是指针类型的错觉。

从上例中还可以看出`std::nullptr_t *`等类型是指针类型，它们是指向`std::nullptr_t`的指针类型。


### 原理
`std::nullptr_t`不是指针类型，但是却可以转换成任意指针类型，并且以0为值。
它的一种实现方式如下：

```C++
#ifdef _LIBCPP_HAS_NO_NULLPTR

_LIBCPP_BEGIN_NAMESPACE_STD

struct _LIBCPP_TEMPLATE_VIS nullptr_t
{
    void* __lx;

    struct __nat {int __for_bool_;};

    _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR nullptr_t() : __lx(0) {}
    _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR nullptr_t(int __nat::*) : __lx(0) {}

    _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR operator int __nat::*() const {return 0;}

    template <class _Tp>
        _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR
        operator _Tp* () const {return 0;}

    template <class _Tp, class _Up>
        _LIBCPP_INLINE_VISIBILITY
        operator _Tp _Up::* () const {return 0;}

    friend _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR bool operator==(nullptr_t, nullptr_t) {return true;}
    friend _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR bool operator!=(nullptr_t, nullptr_t) {return false;}
};

inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR nullptr_t __get_nullptr_t() {return nullptr_t(0);}

#define nullptr _VSTD::__get_nullptr_t()

_LIBCPP_END_NAMESPACE_STD

#else  // _LIBCPP_HAS_NO_NULLPTR

namespace std
{
    typedef decltype(nullptr) nullptr_t;
}

#endif // _LIBCPP_HAS_NO_NULLPTR
```

可见`std::nullptr_t`不是指针类型，可能是类似类的类型。
所以当然可以构造出指向`std::nullptr_t`的指针类型了，例如`std::nullptr_ *`，`const std::nullptr_ *`等。
那么`std::nullptr_t`是类类型吗？


### nullptr是类类型吗？
通过以上源码可以看出`std::nullptr_t`通过宏开关设置了两种实现方式，
第一种就是一个普通的struct，而第二种似乎是编译器内置的，看不出类型。

判断一个类型是否是类类型，可用std::is_class。
测试代码如下：

```C++
watch(std::is_class_v<std::nullptr_t>,
    std::is_union_v<std::nullptr_t>,
    std::is_null_pointer_v<std::nullptr_t>,
    std::is_scalar_v<std::nullptr_t>,
    std::is_object_v<std::nullptr_t>);

watch(std::is_null_pointer_v<std::nullptr_t>,
    std::is_null_pointer_v<void*>);
```

```Txt
std::is_class_v<std::nullptr_t> = 0
std::is_union_v<std::nullptr_t> = 0
std::is_null_pointer_v<std::nullptr_t> = 1
std::is_scalar_v<std::nullptr_t> = 1
std::is_object_v<std::nullptr_t> = 1

std::is_null_pointer_v<std::nullptr_t> = 1
std::is_null_pointer_v<void*> = 0
```

可见`std::nullptr_t`也不是类类型。它就是独特的一个类型，可用`std::is_null_pointer`来判断。


## 如何用C++的方式把\"T*\"转换成\"void*\"

假如`T`是某种未知的模版类型，如何把`T*`的变量转换成`void*`类型呢？
用C语言的强制类型转换很容易实现：
```C++
template <typename T>
void* c_cast(T* p) { // 使用C风格的强制类型转换
    return (void*)(p);
}
```

那么如何用C++的类型转换方式实现呢？

实时上只用reinterpret_cast或static_cast是不行的，因为它们不能将 cv-qualified pointer 转换成
cv-unqualified pointer。
必须首先使用const_cast将cv属性去掉，然后再使用reinterpret_cast或static_cast。如下：

```C++
template <typename T>
void* cpp_cast(T* p) { // 使用C++风格的强制类型转换
    // return reinterpret_cast<void*>(p); // error
    return reinterpret_cast<void*>(const_cast<std::remove_cv_t<T>*>(p)); // OK
    // 或者用static_cast亦可
    // return static_cast<void*>(const_cast<std::remove_cv_t<T>*>(p)); // OK
}
```

## cv-qualified std::nullptr_t及其指针

经过以上讨论，知道std::nullptr_t不是指针类型，那么如下两个问题就可迎刃而解。

### cv-qualified std::nullptr_t 类型的变量能直接赋值给 std::nullptr_t 类型的变量吗？

可以。

```C++
const volatile std::nullptr_t p{0};
std::nullptr_t np = p; // OK
```

### cv-qualified std::nullptr_t * 类型的变量能直接赋值给 std::nullptr_t * 类型的变量吗？

不可以。需要用const_cast转换后方可赋值。

```C++
const volatile std::nullptr_t * p{0};
// std::nullptr_t* np = p; // Error
std::nullptr_t* np = const_cast<std::nullptr_t*>(p); // OK
```
