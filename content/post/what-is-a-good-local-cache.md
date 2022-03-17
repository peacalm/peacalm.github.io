---
title: "What Is a Good Local Cache"
date: 2022-03-17T21:15:21+08:00
draft: true
---

# 灵活好用

* 动态设置每一个key的过期时间
* 过期数据不自动删除，可配置callback on expire
* 支持多种query flags:
  * uncacheable:
  * refresh:
  * peek:
  * stale:
  * fast:
  * preload:


# 高性能
* 并发性好。例如采用分桶机制
* 内存使用效率高

