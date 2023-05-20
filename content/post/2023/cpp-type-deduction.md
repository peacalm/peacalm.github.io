---
title: "查验C++类型推导结果 | Check C++ Type Deduction Results"
date: 2023-05-16T00:25:39+08:00
lastmod: 2023-05-20T00:25:39+08:00
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
是没有成员函数引用属性还是具有成员函数左值引用或成员函数右值引用
（**请注意，这里一定是成员函数引用才算作函数类型签名，C语言函数的引用在此不算作函数类型**），
还有C++17以后一个函数是否是noexcept也认作不同的函数类型，
所以根据这五种属性可以算出总共是`2 * 2 * 2 * 3 * 2 = 48`种！

判断一个类型是否是函数类型，C++标准库中定义了`std::is_function`来实现这个判断。
它的一种可能的实现方式就是把以上所说所有的函数类型形式都特化一遍，如下所示：
```C++
// primary template
template<typename>
struct is_function : std::false_type { };
 
// specialization for regular functions
template<typename Return, typename... Args>
struct is_function<Return(Args...)> : std::true_type {};
 
// specialization for variadic functions such as std::printf
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...)> : std::true_type {};
 
// specialization for function types that have cv-qualifiers
template<typename Return, typename... Args>
struct is_function<Return(Args...) const> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) volatile> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const volatile> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) volatile> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const volatile> : std::true_type {};
 
// specialization for function types that have ref-qualifiers
template<typename Return, typename... Args>
struct is_function<Return(Args...) &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) volatile &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const volatile &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) volatile &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const volatile &> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) volatile &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const volatile &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) volatile &&> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const volatile &&> : std::true_type {};
 
// specializations for noexcept versions of all the above (C++17 and later)
 
template<typename Return, typename... Args>
struct is_function<Return(Args...) noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) volatile noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const volatile noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) volatile noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const volatile noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) volatile & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const volatile & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) volatile & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const volatile & noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) volatile && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args...) const volatile && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) volatile && noexcept> : std::true_type {};
template<typename Return, typename... Args>
struct is_function<Return(Args..., ...) const volatile && noexcept> : std::true_type {};
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
以上48种函数类型形式其实可分成两类，
第一类是**C语言函数形式**的函数类型，上文也提到，C++规定它们不允许有const和volatile属性，
当然它不是成员函数自然也不能有成员函数引用属性，只能有可变长实参属性和noexcept属性；
剩下的就是第二类，**非C语言函数形式**的函数类型，凡是具有const属性、volatile属性、
成员函数引用属性的都属于这类，这一类更像是成员函数，但是不具有母类类型信息，不是完整的成员函数类型，
姑且叫**成员函数形式**的函数类型吧。

用cppreference的说法，也可以说成第一类是**not cv- or ref- qualified** function type，
第二类是**cv- or ref- qualified** function type.
不过也许说成C语言函数形式和成员函数形式，理解它们一个是完整的，一个是不完整的，
可能能更好得理解它们为什么会有以下令人费解令人诧异的不同表现。

算一下，第一类有4种，第二类有44种。

### add reference and remove reference
针对C语言函数形式的函数类型添加引用结果如下：
```C++
CETYPE(std::add_lvalue_reference_t<void(int)>);                // void (&)(int)
CETYPE(std::add_rvalue_reference_t<void(int)>);                // void (&&)(int)
CETYPE(std::add_lvalue_reference_t<void(int, ...)>);           // void (&)(int, ...)
CETYPE(std::add_rvalue_reference_t<void(int, ...)>);           // void (&&)(int, ...)
CETYPE(std::add_lvalue_reference_t<void(int) noexcept>);       // void (&)(int) noexcept
CETYPE(std::add_rvalue_reference_t<void(int) noexcept>);       // void (&&)(int) noexcept
CETYPE(std::add_lvalue_reference_t<void(int, ...) noexcept>);  // void (&)(int, ...) noexcept
CETYPE(std::add_rvalue_reference_t<void(int, ...) noexcept>);  // void (&&)(int, ...) noexcept

// 引用折叠
CETYPE(std::add_lvalue_reference_t<void (&)(int)>);   // void (&)(int)
CETYPE(std::add_rvalue_reference_t<void (&)(int)>);   // void (&)(int)
CETYPE(std::add_lvalue_reference_t<void (&&)(int)>);  // void (&)(int)
CETYPE(std::add_rvalue_reference_t<void (&&)(int)>);  // void (&&)(int)
```
可见这些函数类型都被添加了引用，不过应该注意这个是常规引用，不是成员函数引用，这个引用不影响函数签名。
顺便提一下，对以上输出的带引用的类型施加std::remove_reference_t当然也会回到原输入类型。

针对成员函数形式的函数类型add和remove引用结果如下：
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
可见无论是add还是remove reference，对输入的函数类型，都没有做任何更改。所以可得：

* C语言函数形式的函数类型可以添加引用，而成员函数形式的函数类型本身自带的引用属性不会被更改。

成员函数的引用，在这里由于成为了函数签名的一部分，属于函数类型的一部分，所以这时的引用就与普通常规引用大不相同，
似乎比普通引用的地位更高一等，是不能通过std::add_lvalue_reference, std::remove_reference等被改变的。

那么这类引用就太特殊了，凭什么都是函数类型，一些能被添加引用和去除引用，而另一种不能呢？
是否应该把它定义为另外一类引用，或者另外起一个名字称为另外一类功能？
然后与常规引用不冲突，再可以添加常规引用呢？那么可能就会出现形如`void (&)(int) const &&`
之类的东西了。😂复杂程度又增加了，不过好像又能够和第一类函数类型保持一致了，是好还是不好呢？
C++的选择是不这么做，否则的话，还要再提供一种成员引用的功能与之匹配。C++现在已经有一种
成员指针的功能，即Pointer-to-member operator `.*`和`->*`，这已经够用了。

其实不仅成员函数，数据成员也是不能添加引用的，只能用成员指针（再配合std::mem_fn）。

### add pointer, add const/volatile, remove const/volatile
```C++
// C语言函数形式，能添加指针
CETYPE(std::add_pointer_t<void(int)>);              // void (*)(int)
CETYPE(std::add_pointer_t<void(int,...)>);          // void (*)(int, ...)
CETYPE(std::add_pointer_t<void(int,...) noexcept>); // void (*)(int, ...) noexcept

// 成员函数形式，不能添加指针
CETYPE(std::add_pointer_t<void(int) const>);       // void (int) const
CETYPE(std::add_pointer_t<void(int,...) &>);       // void (int, ...) &
CETYPE(std::add_pointer_t<void(int) volatile &&>); // void (int) volatile &&

// C语言函数形式，不能添加cv
CETYPE(std::add_const_t<void(int)>);           // void (int)
CETYPE(std::add_volatile_t<void(int,...)>);    // void (int, ...)
CETYPE(std::add_cv_t<void(int,...) noexcept>); // void (int, ...) noexcept

// 成员函数形式，不能更改原有cv
CETYPE(std::add_const_t<void(int) & noexcept>);   // void (int) & noexcept
CETYPE(std::add_volatile_t<void(int,...) const>); // void (int, ...) const
CETYPE(std::add_cv_t<void(int) volatile &&>);     // void (int) volatile &&
CETYPE(std::remove_cv_t<void(int) const>);             // void (int) const
CETYPE(std::remove_cv_t<void(int,...) volatile &>);    // void (int, ...) volatile &
CETYPE(std::remove_cv_t<void(int) const && noexcept>); // void (int) const && noexcept
```
可见：
* C语言函数形式的函数类型可以添加指针，而成员函数形式的函数类型不能添加指针；
* C语言函数形式的函数类型不能添加cv属性，而成员函数形式的函数类型本身自带的cv属性也不能被更改。

C语言函数形式的函数类型是完整的函数类型，但成员函数形式的函数类型是不完整的，
因为它不具有包含它的母类的类型信息，所以不难理解它不能被添加指针。
实际上，如果强制为其添加指针，会导致编译报错，std::add_pointer_t里面规避了这点，
对于不能添加指针的类型，返回了输入类型本身。

由此可见，**对于函数类型来讲，如果某种属性是属于函数签名的一部分，那么这种属性就是这个基本类型的一部分，它是不会被改变的。**

用std::is_const, std::is_references校验一下：
### is_reference, is_const, is_volatile
```C++
std::cout << std::boolalpha;

// 成员函数形式，这些都输出 false
std::cout << std::is_reference_v<void(int) &> << std::endl;
std::cout << std::is_reference_v<void(int) &&> << std::endl;
std::cout << std::is_reference_v<void(int) const &> << std::endl;
std::cout << std::is_const_v<void(int) const > << std::endl;
std::cout << std::is_const_v<void(int) const &> << std::endl;
std::cout << std::is_volatile_v<void(int) volatile > << std::endl;
std::cout << std::is_reference_v<void(int)> << std::endl;
std::cout << std::is_const_v<void(int)> << std::endl;
std::cout << std::is_volatile_v<void(int)> << std::endl;

// C语言函数形式，这些都输出 true
std::cout << std::is_reference_v<void (&)(int) > << std::endl;
std::cout << std::is_reference_v<void (&&)(int)> << std::endl;
std::cout << std::is_reference_v<void (&)(int) noexcept> << std::endl;
```
可见当const、volatile和引用等属性成为函数签名即成为函数类型的一部分的时候，这种属性就不能用
std::is_reference, std::is_const, std::is_volatile判断出来了，结果永远是false。

## 如何判断一个类型是否是 std::function
std::function是对函数的封装，它接受一个模板参数，该模版参数应该是一个函数类型。
经过以上讨论，我们知道函数类型有48种形式，那么是否每一种std::function都支持呢？
对于成员函数形式的函数类型我们知道它们是不完整的，对这些函数类型特化没有任何意义，那剩下的4种
C语言函数形式的函数类型 std::function是否都支持呢？事实是，目前（2023）std::function
只支持最基本的那一种，也就是只有这一种特化：
```C++
template< class > class function; /* undefined */   
template< class R, class... Args > class function<R(Args...)>; 
// 没有对其他形式特化
```
验证举例：
```C++
std::function<void(int)> f1; // OK!

// 这些都会编译错误：implicit instantiation of undefined template。因为标准库中并没有为其特化。
// std::function<void(int, ...)> f2;
// std::function<void(int) noexcept> f3
// std::function<void(int) const> f4;
// std::function<void(int) &> f5;
// std::function<void(int) &> f6;
```
上例中f2~f6会编译错误，说明std::function不支持这些函数类型。
当然如果只声明f2~f6对应的那些类型而不用它们定义变量，是不会报错的，但是这没有意义，
这只是声明了一个没有任何实现的类类型而已。

所以目前std::function只支持这一种最基本的函数类型形式，
**要想判断一个类型是否是std::function类型，只需要对`Return(Args...)`这一种函数类型形式做一个特化即可。**
```C++
// 只需这一个primary template和一个特化
template <typename T>
struct is_stdfunction : std::false_type {};
template <typename Return, typename... Args>
struct is_stdfunction<std::function<Return(Args...)>> : std::true_type {};

// 以下这些特化都是不需要的，因为std::function的实现中并没有为其特化
// template <typename Return, typename... Args>
// struct is_stdfunction<std::function<Return(Args..., ...)>> : std::true_type {};
// template <typename Return, typename... Args>
// struct is_stdfunction<std::function<Return(Args...) noexcept>> : std::true_type {};
// template <typename Return, typename... Args>
// struct is_stdfunction<std::function<Return(Args..., ...) noexcept>> : std::true_type {};
```
### std::function如何处理可变长实参(variadic functions)
此外，还需要说明的一个特例是可变长实参，std::function可以说是不支持这种函数类型，但也可以说是
不完全支持，或部分支持。

std::function的特化中虽然不支持variadic这种函数类型形式，
但是事实上我们可以将带有可变长实参的函数当成没有可变长实参的函数来用，
然后再封装在std::function里面，不过这样虽然可行，但是却丢了可变长实参部分的功能。
例如：
```C++
// std::function<int(const char*, ...)> f1 = printf;  // error: implicit instantiation of undefined template 'std::function<int (const char *, ...)>'
std::function<int(const char*)> f2 = printf; // OK
f2("printf no more arguments"); // OK
// f2("printf %d", 1);  // error: no matching function for call to object of type 'std::function<int (const char *)>'
```
也许这也没有多大实用性，所以严格来说，std::function对带有可变长实参的函数类型还是不支持的。
那么应不应该支持一下呢？未来std::function会不会对带有variadic和noexcept的函数类型做一个特化呢？
如果他想，其实是可以的，那时候我们的is_stdfunction的后三种特化就得用上了。
不过，对noexcept的特化其实是不必要的，因为带有noexcept的函数指针，是可以转化成对应不带noexcept的函数指针的。
例如：
```C++
void f1(int) {}
void f2(int) noexcept {}
int main() {
    static_cast<decltype(&f1)>(&f2);  // OK
    // static_cast<decltype(&f2)>(&f1);  // error

    auto p1 = &f1;
    p1 = &f2;  // OK
    
    auto p2 = &f2;
    // p2 = &f1;  // error
}
```
例中也看到了，反过来，不带有noexcept的函数指针是不能转化成对应的带有noexcept的函数指针的。
这也可以理解，毕竟noexcept是更严格的限制，可是说是子集，而不带noexcept的是超集。

再有，针对可变长实参这个属性呢？
当然，有可变长实参的函数指针与对应的没有可变长实参函数指针之间是不能相互转化的，例如：
```C++
void f1(int) {}
void f2(int, ...) {}
int main() {
    // static_cast<decltype(&f1)>(&f2);  // error
    // static_cast<decltype(&f2)>(&f1);  // error
    
    auto p1 = &f1;
    // p1 = &f2;  // error
    
    auto p2 = &f2;
    // p2 = &f1;  // error
}
```
这个std::function不对其特化，可能就是不想支持这种函数类型了吧，毕竟这是C语言遗留产物。


## 如何判断一个类型是否是 lambda
lambda是C++中的匿名类型，每个新定义的lambda都具有各自不同的类型，一般来说我们无法判断一个类型是否是lambda。

但是有一种特殊的lambda，即捕获列表为空的（captureless）且不是generic的
（使用auto声明参数类型的叫作generic lambda），这种特殊的lambda有一个独有的性质，
就是它能够转换成与它的operator()对应的C语言风格的函数指针，并且指向它的operator()。

我们可以判断一个类型是否具有这种性质，假如具有，那么我们可以猜测这种类型大概率可能就是
这种特殊的lambda，除非用户手动定义了一个具有这种性质的自定义类型，而通常很少有人这么做，也不该这么做。

所以虽然我们无法判断一个类型是否是lambda，但我们可以判断一个类型有可能是
captureless并且non-generic的lambda。

```C++
// Get a C function pointer type by a member function pointer type

template <typename T>
struct detect_cfunction {
  using type = void;
};

template <typename Object, typename Return, typename... Args>
struct detect_cfunction<Return (Object::*)(Args...)> {
  using type = Return (*)(Args...);
};

template <typename Object, typename Return, typename... Args>
struct detect_cfunction<Return (Object::*)(Args...) const> {
  using type = Return (*)(Args...);
};

template <typename Object, typename Return, typename... Args>
struct detect_cfunction<Return (Object::*)(Args..., ...)> {
  using type = Return (*)(Args..., ...);
};

template <typename Object, typename Return, typename... Args>
struct detect_cfunction<Return (Object::*)(Args..., ...) const> {
  using type = Return (*)(Args..., ...);
};

template <typename T>
using detect_cfunction_t = typename detect_cfunction<T>::type;

// Detect type of callee, which is a pointer to member function operator()

template <typename T, typename = void>
struct detect_callee {
  using type = void;
};

template <typename T>
struct detect_callee<T, std::void_t<decltype(&T::operator())>> {
  using type = decltype(&T::operator());
};

template <typename T>
using detect_callee_t = typename detect_callee<T>::type;

// Detect C function style callee type
template <typename T>
using detect_c_callee_t = detect_cfunction_t<detect_callee_t<T>>;

// Detect whether T maybe a captureless and non-generic lambda
template <typename T>
struct maybe_lambda
    : std::integral_constant<
          bool,
          !std::is_same<T, void>::value &&
              std::is_convertible<T, detect_c_callee_t<T>>::value> {};
```

举例：
```C++
#define MAYBE_LAMBDA(x) std::cout << #x << ": " << maybe_lambda<x>::value << std::endl;

int main() {
    std::cout << std::boolalpha;
    
    // 这两个为 true
    auto l1 = [](int){};
    MAYBE_LAMBDA(decltype(l1));
    auto l2 = [](int, ...){};
    MAYBE_LAMBDA(decltype(l2));
    
    // 以下均为 false
    
    auto l3 = [](auto){}; // generic lambda
    MAYBE_LAMBDA(decltype(l3));
    auto l4 = [&](int){};
    MAYBE_LAMBDA(decltype(l4));
    auto l5 = [=](int){};
    MAYBE_LAMBDA(decltype(l5));
    
    MAYBE_LAMBDA(std::function<void(int)>);
    MAYBE_LAMBDA(int);
    MAYBE_LAMBDA(void);
    MAYBE_LAMBDA(int*);
    MAYBE_LAMBDA(void*);
    
    return 0;
}
```
输出结果如下：
```Txt
decltype(l1): true
decltype(l2): true
decltype(l3): false
decltype(l4): false
decltype(l5): false
std::function<void(int)>: false
int: false
void: false
int*: false
void*: false
```