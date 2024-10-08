---
title: "C++轻量级输出库MyOStream：可打印输出所有成员可迭代的容器"
date: 2022-03-17T01:12:25+08:00
lastmod: 2022-03-18T00:54:42+08:00
draft: false
tags: ["C++", "库", "输出库", "C++实现", "debug tool", "watch tool", "ACMer助手", "开源"]
categories: ["技术"]
---

## 懒汉的烦恼

使用C++编程时对数据打印输出比较麻烦，需要自行用for循环将vector, list, map等容器的成员一一打印输出。
相比之下Python, Golang等语言就可以直接对所有数据类型打印输出，这对于debug是很友好的特性。
因此，我开发了一个简单的C++库，几乎能够对所有容器直接打印输出，说几乎是因为我们只能对成员可访问可迭代的容器支持这个特性。

支持的类型如下，以及他们的组合类型：

std::pair, std::tuple, std::array, std::deque, 
std::forward_list, std::initializer_list, std::list, std::vector, 
std::set, std::multiset, std::unordered_set, std::unordered_multiset,
std::map, std::multimap, std::unordered_map, std::unordered_multimap.

## ACMer助手

特别的，我们在打ACM比赛或做类似的OJ题目的过程中，需要debug的时候，我们不仅想要打印容器里的值，
而且还想要同时打印出变量名，以便我们在解题时，如果定义了多个容器变量，我们可以知道哪个变量的值使哪一个值，
因此我设计了watch功能。

宏定义watch可以用一下两种方式：
一，用std::cout输出MYOSTREAM_WATCH_TO_STRING生成的字符串：
```C++
#include <myostream.h>
#define watch(...) std::cout << MYOSTREAM_WATCH_TO_STRING(std::string, " = ", "\n", "\n\n", __VA_ARGS__)
```
二，直接使用MYOSTREAM_WATCH，坏处是这里定义了一个变量`mycout`，这段代码不能在头文件中被多个源文件include:
```C++
#include <myostream.h>
myostream::ostream mycout(std::cout.rdbuf());
#define watch(...) MYOSTREAM_WATCH(mycout, " = ", "\n", "\n\n", __VA_ARGS__)
```

例如，如下debug代码：
```C++
#include <iostream>
#include <vector>

#ifndef ONLINE_JUDGE
#include <myostream.h>
myostream::ostream mycout(std::cout.rdbuf());
#define watch(...) MYOSTREAM_WATCH(mycout, " = ", "\n", "\n\n", __VA_ARGS__)
#else
#define watch(...)
#endif

int main() {
  std::vector<int> v{1, 2, 3};
  watch(v);

  std::tuple<std::vector<int>, std::map<int, std::set<std::string>>,
      std::pair<std::string, double>> complex({1, 2, 3},
                                              {{1, {"a", "aa"}},
                                               {2, {"b", "bb"}}},
                                              {"Aya", 88.5});
  watch(complex);

  return 0;
}
```

在本地编译运行，可以输出：
```text
v = [1, 2, 3]

complex = <[1, 2, 3], {1: {a, aa}, 2: {b, bb}}, (Aya, 88.5)>
```
而在线提交，或定义了宏ONLINE_JUDGE后，则什么都不输出。

## 附录

* [lib MyOStream on github](https://github.com/peacalm/myostream)
