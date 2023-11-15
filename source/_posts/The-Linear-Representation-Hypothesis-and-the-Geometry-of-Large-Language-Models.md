---
title: The Linear Representation Hypothesis and the Geometry of Large Language Models
date: 2023-11-15 22:30:33
mathjax: true
tags:
- Large Language Models
---

本文是关于文章 [The Linear Representation Hypothesis and the Geometry of Large Language Models](https://arxiv.org/abs/2311.03658) 的笔记.

这篇文章的思想比较直白, 无外乎以下三点.

+ 每个概念(的出现与否)都被表达为(上下文/词)向量在一个子空间上的投影的取值.
+ 进一步地, 通过对这样的子空间使用线性的探针, 可以测量概念的出现.
+ 最后, 通过在该子空间上使用线性的一些操作, 可以干预概念的取值.

<!-- more -->

回顾自回归 LLM 预测下文的方式.

LLM 首先将上文 $x$ 映射到嵌入向量 $$\lambda(x)\in \Lambda \simeq \mathbb{R}^d$$

LLM 还将每个词(token) $y$ 表达为反嵌入向量 $$\gamma(y)\in \Gamma \simeq \mathbb{R}^d$$

LLM 预测的则是上文 $X=x$ 时下文 $Y=y$ 的概率, 即 $$P(Y=y\vert X=x)\propto \exp(\lambda(x)^\top \gamma(y))$$

概念变量 $W$ 则是一个隐变量, 其被上文$X$决定, 并干预模型的输出$Y$: $$Y\vert_{W=w}:=Y(W=w)$$

我们可以对$W$生成一个集合. 例如 $$(Y(W=1), Y(W=0))\in \{(\text{man}, \text{woman}), (\text{king}, \text{queen}), \dots\}$$ 是对于男女二元性别这个概念生成的集合

概念变量的表征才是我们实际能观测和干预的东西. 如果表征空间$\Gamma$中的向量$\bar{\gamma_W}$满足 $$\gamma(Y(W=1))-\gamma(Y(W=0)) \in \text{Cone}(\bar{\gamma_W})$$

// TODO
