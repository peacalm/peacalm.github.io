---
title: "Maglev Consistent Hash"
date: 2022-03-31T18:33:57+08:00
lastmod: 2022-03-31T18:33:57+08:00
draft: true
tags: ["maglev", "一致性哈希"]
categories: ["技术"]
---

## Maglev一致性哈希算法描述
1. 选取哈希槽长度，素数M，生成空哈希槽；
2. 将server列表排序；
3. 对每一个server哈希得到两个随机数A、B(模M-1再加1保证非0或M的倍数)，然后得到server对应的0到M-1的排列：Xk = (A + kB)%M, k=0,1,...,M-1；
4. 排列中每一个数字代表一个槽位，轮询每一个server的排列，从左到右选择排列中的第一个空槽位填充进去，直到哈希槽填充完整为止；
5. 对一个query_id，query_id%M 映射到哈希槽中对应的server。

## 算法优缺点
* 优点：分片均匀，查询速度快O(1)
* 缺点：更新较慢O(n)
