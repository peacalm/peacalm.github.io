---
title: "查看C++类型推导结果 | Check C++ Type Deduction Results"
date: 2023-05-16T00:25:39+08:00
lastmod: 2023-05-17T00:25:39+08:00
draft: false
tags: ["C++", "类型推导", "泛型"]
categories: ["技术"]
#math: false
#toc: false
---

## 如何查看类型

C++的类型系统是极其复杂的，基本类型与const, volatile, 指针，引用，数组，函数，类，成员变量，
成员函数等特性的组合能生成许多不同的类型。类型推导是C++泛型编程/模版元编程的基础。
那么如何查看某个类型或变量具体是什么类型呢？我使用的方法有
* C++内置的typeid
* boost::typeindex::type_id
* 猜测类型并用std::is_same校验
* 故意使编译报错在编译器给出的错误信息中查看目标的类型

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

事实上，C++内置的typeid和Boost的type_id得出的结果都是对输入类型去掉了
const, volatile和引用属性的。

那有没有获取完整类型的方法呢？
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
写某种会编译报错的代码，让我们想要查看的类型名展示在报错信息中：
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
```txt
error: no member named 'NX' in 'EMPTY<const int[3]>'
```
于是可以看出a的类型是"const int[3]"。
（也有编译器输出"int const[3]"，这是等效的写法，它们是相同的类型）


## 类型推导示例

### C function
预先定义好的用于类型推导的函数：
```C++
void func(int) {}
```
```C++
{
    CETYPE(decltype(func));  // void (int)
    CETYPE(decltype(&func)); // void (*)(int)

    // implicitly convert to pointer if assigned by function name
    auto fcopy = func;
    CETYPE(decltype(fcopy)); // void (*)(int)
    const auto cfcopy = func;
    CETYPE(decltype(cfcopy)); // void (*const)(int)

    // same as above
    auto fptr = &func;
    CETYPE(decltype(fptr)); // void (*)(int)
    const auto cfptr = &func;
    CETYPE(decltype(cfptr)); // void (*const)(int)

    // static_cast on rvalue reference to function returns lvalue
    CETYPE(decltype(std::move(func)));  // void (&)(int)

    CETYPE(decltype(&std::move(func))); // void (*)(int)
    CETYPE(decltype(std::move(&func))); // void (*&&)(int)

    decltype(func) && prfunc = func;
    CETYPE(decltype(prfunc));  // void (&&)(int)
}

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

    // ref of move
    const auto & clref_of_move = std::move(func);
    CETYPE(decltype(clref_of_move));  // void (&)(int)
    auto && rref_of_move = std::move(func);
    CETYPE(decltype(rref_of_move));   // void (&)(int)
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

定义如下两个模版函数，其参数声明分别是C++11以前的万能引用和C++11引入的新的万能引用。
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
将函数func用作以上两个模版函数的参数，类型推导结果如下：
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
    larg(std::move(func));
    // template<typename T> void larg(const T & t) {
    //     CETYPE(T);           // void (int)
    //     CETYPE(decltype(t)); // void (&)(int)
    // }

    rarg(std::move(func));
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

特别的，std::move作用于函数类型时返回的是lvalue `FunctionType&`而不是`FunctionType&&`。

另外，由于C++规定不是成员函数的函数不能有const、valitale属性，所以
`const auto & clref = func;`得到clref的类型是`void (&)(int)`，const属性被忽略了。
如果换成其他类型，则const属性会被保留，例如int，如下：

### Int
```C++
int i = 0;
// const被保留
const auto & clref = i;
CETYPE(decltype(clref));  // const int &
CETYPE(decltype(&clref)); // const int *
```

```C++
int i = 0;
const int ci = 0;

CETYPE(decltype(i));  // int
CETYPE(decltype(ci)); // const int

// 非单个标记符的左值，推导为T&。例如加一个小括号，或前置自增
CETYPE(decltype((i)));  // int &
CETYPE(decltype((ci))); // const int &
CETYPE(decltype(++i));  // int &

// 后置自增是一个右值，于是推导为T
CETYPE(decltype(i++));  // int

// 右值
CETYPE(decltype(i + ci));  // int

// 右值引用
CETYPE(decltype(std::move(i)));  // int &&
```

```C++
larg(i);
// template<typename T> void larg(const T & t) {
//     CETYPE(T);           // int
//     CETYPE(decltype(t)); // const int &
// }

rarg(i);
// template<typename T> void rarg(T && t) {
//     CETYPE(T);           // int &
//     CETYPE(decltype(t)); // int &
// }

larg(ci);
// template<typename T> void larg(const T & t) {
//     CETYPE(T);           // int
//     CETYPE(decltype(t)); // const int &
// }

rarg(ci);
// template<typename T> void rarg(T && t) {
//     CETYPE(T);           // const int &
//     CETYPE(decltype(t)); // const int &
// }

larg(std::move(i));
// template<typename T> void larg(const T & t) {
//     CETYPE(T);           // int
//     CETYPE(decltype(t)); // const int &
// }

rarg(std::move(ci));
// template<typename T> void rarg(T && t) {
//     CETYPE(T);           // const int
//     CETYPE(decltype(t)); // const int &&
// }

rarg(3);
// template<typename T> void rarg(T && t) {
//     CETYPE(T);           // int 
//     CETYPE(decltype(t)); // int &&
// }

```

### C array
```C++
char a[3];

CETYPE(decltype(a));     // char[3]
CETYPE(decltype(&a));    // char (*)[3]
CETYPE(decltype(&a[0])); // char *

const auto & acr = a;
CETYPE(decltype(acr));  // const char (&)[3]
const auto & pacr = &a;
CETYPE(decltype(pacr)); // char (*const &)[3]
const auto & pa0cr = &a[0];
CETYPE(decltype(pa0cr)); // char *const &

auto && arr = a;
CETYPE(decltype(arr));  // char (&)[3]
auto && parr = &a;
CETYPE(decltype(parr)); // char (*&&)[3]
auto pa = &a;
auto && lparr = pa;
CETYPE(decltype(lparr)); // char (*&)[3]

auto && pa0rr = &a[0];
CETYPE(decltype(pa0rr)); // char *&&
auto pa0 = &a[0];
auto && lpa0rr = pa0;
CETYPE(decltype(lpa0rr)); // char *&
```

```C++
const char a[3];

CETYPE(decltype(a));     // const char[3]
CETYPE(decltype(&a));    // const char (*)[3]
CETYPE(decltype(&a[0])); // const char *

const auto & acr = a;
CETYPE(decltype(acr));  // const char (&)[3]
const auto & pacr = &a;
CETYPE(decltype(pacr)); // const char (*const &)[3]
const auto & pa0cr = &a[0];
CETYPE(decltype(pa0cr)); // const char *const &

auto && arr = a;
CETYPE(decltype(arr));  // const char (&)[3]
auto && parr = &a;
CETYPE(decltype(parr)); // const char (*&&)[3]
auto pa = &a;
auto && lparr = pa;
CETYPE(decltype(lparr)); // const char (*&)[3]

auto && pa0rr = &a[0];
CETYPE(decltype(pa0rr)); // const char *&&
auto pa0 = &a[0];
auto && lpa0rr = pa0;
CETYPE(decltype(lpa0rr)); // const char *&
```

```C++
int ia[] = {};
int ia2[] = {1, 2};
int ia3[3] = {1};

CETYPE(decltype(ia));  // int[0]
CETYPE(decltype(ia2)); // int[2]
CETYPE(decltype(ia3)); // int[3]
CETYPE(int[]);         // int[]
CETYPE(const int[]);   // const int[]
CETYPE(decltype(&ia)); // int (*)[0]
```

### C string
字面字符串常量，decltype推断结果如下：
```C++
CETYPE(decltype("abcd")); // const char (&)[5]

auto s = "abcd";
CETYPE(decltype(s)); // const char *
```

## C++中的函数类型到底有多少种形式？
影响函数类型的修饰属性都是正交的，因此把每一种属性的选项数量乘起来就是答案。
根据一个函数类型是否是const的，是否是volatile的，是否有C语言风格的可变长实参，
是没有成员函数引用属性还是具有成员函数左值引用或成员函数右值引用，
**请注意，这里一定是成员函数引用才算作函数类型签名，C语言函数的引用在此不算作函数类型**，
还有C++17以后一个函数是否是noexcept也认作不同的函数类型，
可以算出总共是`2 * 2 * 2 * 3 * 2 = 48`种！

判断一个类型是否是函数类型，C++标准库中定义了`std::is_function`来实现这个判断。
它的一种可能得实现方式就是把以上所说所有的函数类型形式都特化了一遍，如下所示：
```C++
// primary template
template<typename>
struct is_function : std::false_type { };
 
// specialization for regular functions
template<typename Ret, typename... Args>
struct is_function<Ret(Args...)> : std::true_type {};
 
// specialization for variadic functions such as std::printf
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...)> : std::true_type {};
 
// specialization for function types that have cv-qualifiers
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) volatile> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const volatile> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) volatile> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const volatile> : std::true_type {};
 
// specialization for function types that have ref-qualifiers
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) volatile &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const volatile &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) volatile &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const volatile &> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) volatile &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const volatile &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) volatile &&> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const volatile &&> : std::true_type {};
 
// specializations for noexcept versions of all the above (C++17 and later)
 
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) volatile noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const volatile noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) volatile noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const volatile noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) volatile & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const volatile & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) volatile & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const volatile & noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) volatile && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args...) const volatile && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) volatile && noexcept> : std::true_type {};
template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...) const volatile && noexcept> : std::true_type {};
```

针对上文特别提起的**引用属性**举个例子：
```C++
// 这两个是C语言函数的引用，判断为false
std::is_function<void(&)(int)>::value;
std::is_function<void(&&)(int)>::value;

// 而这两个是成员函数的引用，判断为true
std::is_function<void(int) &>::value;
std::is_function<void(int) &&>::value;
```
## 对函数类型进行add pointer/c/v/r, remove c/v/r等操作是什么结果？
以上48种函数类型形式其实可分成两类，第一类是非成员函数类型形式，也就是C语言函数类型形式，
上文也提到，C++规定它们不允许有const和volatile属性，当然也不能有成员函数引用属性，
只能有可变长实参属性和noexcept属性；剩下的就是第二类，即成员函数类型形式，
凡是具有const属性、volatile属性、成员函数引用属性的都属于这类。

### add reference and remove reference
针对C语言函数类型形式的函数类型添加引用结果如下：
```C++
CETYPE(std::add_lvalue_reference_t<void(int)>);                // void (&)(int)
CETYPE(std::add_rvalue_reference_t<void(int)>);                // void (&&)(int)
CETYPE(std::add_lvalue_reference_t<void(int, ...)>);           // void (&)(int, ...)
CETYPE(std::add_rvalue_reference_t<void(int, ...)>);           // void (&&)(int, ...)
CETYPE(std::add_lvalue_reference_t<void(int) noexcept>);       // void (&)(int) noexcept
CETYPE(std::add_rvalue_reference_t<void(int) noexcept>);       // void (&&)(int) noexcept
CETYPE(std::add_lvalue_reference_t<void(int, ...) noexcept>);  // void (&)(int, ...) noexcept
CETYPE(std::add_rvalue_reference_t<void(int, ...) noexcept>);  // void (&&)(int, ...) noexcept
```
可见这些函数类型都被添加了引用，不过应该注意这个是常规引用，不是成员函数引用，这个引用不影响函数签名。
顺便提一下，对以上输出的带引用的类型施加std::remove_reference_t当然也会回到原输入类型。

针对成员函数类型形式的函数类型add和remove引用结果如下：
```C++
// add reference
CETYPE(std::add_lvalue_reference_t<void(int) const>);              // void (int) const
CETYPE(std::add_rvalue_reference_t<void(int) volatile>);           // void (int) volatile
CETYPE(std::add_lvalue_reference_t<void(int, ...) volatile>);      // void (int, ...) volatile
CETYPE(std::add_rvalue_reference_t<void(int, ...) const>);         // void (int, ...) const
CETYPE(std::add_lvalue_reference_t<void(int) const noexcept>);     // void (int) const noexcept
CETYPE(std::add_rvalue_reference_t<void(int) volatile noexcept>);  // void (int) volatile noexcept
CETYPE(std::add_lvalue_reference_t<void(int) &>);                  // void (int) &
CETYPE(std::add_rvalue_reference_t<void(int) &>);                  // void (int) &
CETYPE(std::add_lvalue_reference_t<void(int) &&>);                 // void (int) &&
CETYPE(std::add_rvalue_reference_t<void(int) &&>);                 // void (int) &&
CETYPE(std::add_lvalue_reference_t<void(int) const &>);            // void (int) const &
CETYPE(std::add_rvalue_reference_t<void(int) volatile &>);         // void (int) volatile &
CETYPE(std::add_lvalue_reference_t<void(int) const && noexcept>);  // void (int) const && noexcept

// remove reference
CETYPE(std::remove_reference_t<void(int) &>);                    // void (int) &
CETYPE(std::remove_reference_t<void(int) &&>);                   // void (int) &&
CETYPE(std::remove_reference_t<void(int) const &>);              // void (int) const &
CETYPE(std::remove_reference_t<void(int) const &&>);             // void (int) const &&
CETYPE(std::remove_reference_t<void(int) volatile &>);           // void (int) volatile &
CETYPE(std::remove_reference_t<void(int) & noexcept>);           // void (int) & noexcept
CETYPE(std::remove_reference_t<void(int) const & noexcept>);     // void (int) const & noexcept
CETYPE(std::remove_reference_t<void(int) volatile && noexcept>); // void (int) volatile && noexcept
```
可见无论是add referenc还是remove reference，对输入的无论哪种函数类型，都没有任何影响。

由此可见，对于函数类型来讲，如果某种属性是属于函数签名的一部分，那么这种属性就是这个基本类型的一部分，它是不会被改变的。成员函数的引用，在这里由于变成了函数签名的一部分，所以这时的引用就与普通常规引用大不相同。然后C++规定，成员函数类型形式的函数类型，是不能添加常规引用的（如上文C语言函数类型形式的被添加的那种），由此可以猜测，那么指针应该也是不能被添加的，而事实确实如此。

### add const/volatile, remove const/volatile, add pointer
由上文中总结的规律可以猜测：
C语言函数形式的函数类型可以添加指针，而成员函数形式的函数类型不能添加指针；
C语言函数形式的函数类型不能添加cv属性，而成员函数形式的函数类型本身自带的cv属性也不能被更改。
事实确实如此：
```C++
// C语言函数类型，能添加指针
CETYPE(std::add_pointer_t<void(int)>);              // void (*)(int)
CETYPE(std::add_pointer_t<void(int,...)>);          // void (*)(int, ...)
CETYPE(std::add_pointer_t<void(int,...) noexcept>); // void (*)(int, ...) noexcept

// 成员函数类型，不能添加指针
CETYPE(std::add_pointer_t<void(int) const>);       // void (int) const
CETYPE(std::add_pointer_t<void(int,...) &>);       // void (int, ...) &
CETYPE(std::add_pointer_t<void(int) volatile &&>); // void (int) volatile &&

// C语言函数类型，不能添加cv
CETYPE(std::add_const_t<void(int)>);           // void (int)
CETYPE(std::add_volatile_t<void(int,...)>);    // void (int, ...)
CETYPE(std::add_cv_t<void(int,...) noexcept>); // void (int, ...) noexcept

// 成员函数类型，不能更改原有cv
CETYPE(std::add_const_t<void(int) & noexcept>);   // void (int) & noexcept
CETYPE(std::add_volatile_t<void(int,...) const>); // void (int, ...) const
CETYPE(std::add_cv_t<void(int) volatile &&>);     // void (int) volatile &&
CETYPE(std::remove_cv_t<void(int) const>);             // void (int) const
CETYPE(std::remove_cv_t<void(int,...) volatile &>);    // void (int, ...) volatile &
CETYPE(std::remove_cv_t<void(int) const && noexcept>); // void (int) const && noexcept
```

