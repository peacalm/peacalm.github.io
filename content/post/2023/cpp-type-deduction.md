---
title: "C++类型推导 | C++ Type Deduction"
date: 2023-05-16T00:25:39+08:00
lastmod: 2023-05-16T00:25:39+08:00
draft: false
tags: ["C++", "类型推导"]
categories: ["技术"]
#math: false
#toc: false
---

# 如何查看类型

C++的类型系统是极其复杂的，基本类型与const, volatile, 指针，引用，数组，函数，成员变量，
成员函数等特性的组合能生成不同的类型。类型推导是C++模版元编程的基础。
那么如何查看某个类型或变量具体是什么类型呢？笔者使用的方法有
* C++内置的typeid
* boost::typeindex::type_id
* 用std::is_same猜测校验
* 故意编译报错在编译器给出的错误信息中查看目标的类型

例如用typeid：
```C++
const int a[3] = {};
std::cout << typeid(decltype(a)).name() << std::endl;
// prints: A3_i
```
可见使用typeid获取的类型名不具有好的可读性。于是可以用boost::typeindex::type_id替代：
```C++
// need include this header
// #include <boost/type_index.hpp>
const int a[3] = {};
std::cout << boost::typeindex::type_id<decltype(a)>().pretty_name() << std::endl;
// prints: int [3]
```
可见虽然结果具有很好的可读性，但是却去掉了const属性，与真实输入类型不符合。

事实上，C++内置的typeid和Boost的type_id得出的结果都是对输入类型进行decay之后的类型，
也就是去掉了引用和const, volatile。

那有没有获取目标的完整类型的方法呢？
C++中可以用std::is_same来判断两个类型是否相同，它没有对输入类型做任何改变。
因此可以猜测一个结果然后用它来判断猜得是否正确：
```C++
if (std::is_same_v<decltype(a), int [3]>) {
    // a has type: int [3]
} else if (std::is_same_v<decltype(a), const int [3]>) {
    // a has type: const int [3]
}
```

但是这样太麻烦了，如果不想猜呢？由于编译器编译阶段是知道变量或表达式的类型的，所以我们可以故意
写某种会编译报错的代码，让我们想要查看的目标类型信息展示在报错信息中：
```C++
template <typename> struct EMPTY{};
// CE: compilation error; NX: not exists
#define CETYPE(type) EMPTY<type>::NX()

int main() {
    const int a[3] = {};
    CETYPE(decltype(a));
}
```
编译报错信息是：
```
error: no member named 'NX' in 'EMPTY<const int[3]>'
```
于是可以看出a的类型是"const int[3]"。
（也有编译器输出"int const[3]"，这是等效的写法，它们是相同的类型）


# 类型推导示例

## C function
预先定义好的用于类型推导的函数：
```C++
void func(int) {}
```
```C++
CETYPE(decltype(func));  // void (int)
CETYPE(decltype(&func)); // void (*)(int)

// references
{
    auto & lref = func;
    CETYPE(decltype(lref));  // void (&)(int)
    CETYPE(decltype(&lref)); // void (*)(int)

    const auto & clref = func;
    // 由于规定非成员函数不能有const属性，所以const被忽略掉
    CETYPE(decltype(clref));  // void (&)(int) 
    CETYPE(decltype(&clref)); // void (*)(int)

    auto && rref = func;
    CETYPE(decltype(rref));  // void (&)(int)
    CETYPE(decltype(&rref)); // void (*)(int)
}

// references of pointer
{
    // &func is a rvalue pointer
    const auto & clref = &func;
    CETYPE(decltype(clref)); // void (*const &)(int)
    auto && rref = &func;
    CETYPE(decltype(rref));  // void (*&&)(int)

    // pfunc is a lvalue pointer
    auto pfunc = &func;  // void (*)(int)
    const auto & clref = pfunc;
    CETYPE(decltype(clref)); // void (*const &)(int)
    auto && rref = pfunc;
    CETYPE(decltype(rref));  // void (*&)(int)
}
```

将函数用作如下模版函数的参数：
```C++
template<typename T> void larg(const T & t) {
    CETYPE(T);
    CETYPE(decltype(t));
}
template<typename T> void rarg(T && t) {
    CETYPE(T);
    CETYPE(decltype(t));
}
```
```C++
{
    larg(func);
    // template<typename T> void larg(const T & t) {
    //     CETYPE(T);           // void (int)
    //     CETYPE(decltype(t)); // void (&)(int)
    // }

    rarg(func);
    // template<typename T> void rarg(T && t) {
    //     CETYPE(T);           // void (&)(int)
    //     CETYPE(decltype(t)); // void (&)(int)
    // }
}
{
    larg(&func);
    // template<typename T> void larg(const T & t) {
    //     CETYPE(T);           // void (*)(int)
    //     CETYPE(decltype(t)); // void (*const &)(int)
    // }

    rarg(&func);
    // template<typename T> void rarg(T && t) {
    //     CETYPE(T);           // void (*)(int)
    //     CETYPE(decltype(t)); // void (*&&)(int)
    // }
}
{
    auto pfunc = &func;  // void (*)(int)
    larg(pfunc);
    // template<typename T> void larg(const T & t) {
    //     CETYPE(T);           // void (*)(int)
    //     CETYPE(decltype(t)); // void (*const &)(int)
    // }

    rarg(pfunc);
    // template<typename T> void rarg(T && t) {
    //     CETYPE(T);           // void (*&)(int)
    //     CETYPE(decltype(t)); // void (*&)(int)
    // }
}
```
可见，函数模版参数`const T & t`与定义引用`const auto & r = ...`推导结果一致，
函数模版参数`T && t`与定义引用`auto && r = ...`推导结果一致。

另外，由于C++规定不是成员函数的函数不能有const、valitale属性，所以
`const auto & clref = func;`得到clref的类型是`void (&)(int)`，const属性被忽略了。
用int作为对比如下：
```C++
int ival = 0;
const auto & clref = ival;
CETYPE(decltype(clref));  // const int &
CETYPE(decltype(&clref)); // const int *
```
