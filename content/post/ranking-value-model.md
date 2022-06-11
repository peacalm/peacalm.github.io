---
title: "排序融合公式 | Ranking Value Model"
date: 2022-06-11T20:00:00+08:00
lastmod: 2022-06-11T20:00:00+08:00
draft: false
tags: ["排序", "融合公式"]
categories: ["技术"]
math: true
---

考虑有限候选多目标融合排序公式，目标个数T，候选个数N。

## 加法融合公式

$$
RankScore = \sum_{i=1}^T w_i * (b_i + t_i)
$$

$t_i$ 为第$i$个目标的目标分，$b_i$ 为第$i$个目标的Bias，一般选用0，$w_i$ 为第$i$个目标的权重。


## 乘法融合公式
$$
RankScore = \prod_{i=1}^T (b_i + \beta_i \cdot t_i) ^ {\alpha_i}
$$

$t_i$ 为第$i$个目标的目标分，$b_i$ 为第$i$个目标的Bias，一般选用1，$\alpha_i$和$\beta_i$ 为调整第$i$个目标权重的参数。


## 目标分预处理

### 归一化 | Normalization
目标分分布不稳定时，可以把目标分归一化线性映射到稳定的[A, B]的区间，一般B取1，A可取0.01或更小或0：

$$
\tilde{t}_i = \frac{ t_i - \min t_i } { \max t_i - \min t_i} \cdot (B - A) + A
$$

其中，
$\max t_i = \max \limits_{1 \le j \le N} t_{ij}$ 为第$i$个目标分在$N$个候选上的分布的最大值，
$\min t_i = \min \limits_{1 \le j \le N} t_{ij}$ 为第$i$个目标分在$N$个候选上的分布的最小值。


### 相对得分
在推荐系统中，不同用户对某一个动作的使用频率或喜好程度不同，则表现为某一个目标分在不同用户上的分布有大小区别，
因此可以用该目标分在当前待排序的有限候选集上的相对得分取代原始得分：

$$
\hat{t}\_i = \frac {t_i} { \overline{t}_i }
$$

其中，
$\overline{t}\_i = \frac { \sum_{j=1}^N t_{ij} } {N}$ 为第$i$个目标分在$N$个候选上的分布的平均值。



## 在融合公式中使用排名
对候选对象按某一个目标分排序，则可得到每一个候选对象在这一个目标上的排名$r_i$，为[1, N]之间的整数。
然后可以直接在融合公式中使用排名，或者将排名归一化后再代入融合公式。

