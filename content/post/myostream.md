---
title: "C++轻量级输出库MyOStream：可打印输出所有成员可迭代的容器"
date: 2022-03-17T01:12:25+08:00
draft: false
tags: ["C++", "库", "输出库", "debug tool", "watch tool", "ACMer助手"]
categories: ["C++"]
---

* [lib MyOStream on github](https://github.com/peacalm/myostream)


使用C++编程时对数据打印输出比较麻烦，需要自行用for循环将vector, list, map等容器的成员一一打印
输出，相比之下Python, Golang等语言就可以直接对所有数据类型打印输出，这对于debug是很友好的特性。
因此，我开发了一个简单的C++库，几乎能够对所有容器直接打印输出，说几乎是因为我们只能对成员可访问可
迭代的容器支持这个特性。


特别的，我们在打ACM比赛或做类似的OJ题目的过程中，需要debug的时候，我们
不仅想要打印容器里的值，而且还想要同时打印出变量名，以便我们在解题时，如果定义了多个容器变量，我们
可以知道哪个变量的值使哪一个值，因此我设计了watch功能。


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
  return 0;
}
```

在本地编译运行，可以输出：
```text
v = [1, 2, 3]
```
而在线提交，或定义了宏ONLINE_JUDGE后，则什么都不输出。
