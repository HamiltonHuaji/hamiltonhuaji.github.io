---
title: Notes on Stochastic Processes
date: 2023-11-25 17:59:18
mathjax: true
tags:
---

本文不是关于随机过程的完整介绍. 但如果你学到滤子(Filtration)而不知道它是干什么用的, 就接着往下读.

<!-- more -->

概率空间 $(\Omega, \mathcal{F}, \mathbb{P})$ 中, $\Omega$ 是底层样本空间. $\mathcal{F}$ 是其事件域. $\mathcal{F}$ 的细分程度则决定了我们在该概率空间中描述可观测事件的粒度. 从 $\mathcal{F}$ 所导出的最细粒度的原子事件出发, 我们可以利用 $\mathcal{F}$ 对补和可列并的封闭性生成出完整的 $\mathcal{F}$.

例如, 考虑丢骰子. 底层样本空间为 $\Omega=\{1,2,3,4,5,6\}$, 而 $\mathcal{F}$ 可以是 $\{\{\}, \{1,3,5\}, \{2,4,6\},\Omega\}$, 这描述了丢骰子结果的奇偶性. 在这样的一个(故意选取的) $\mathcal{F}$ 的定义下, 不可能描述更细的事件, 尽管我们实际上知道这些事件都是由多个底层样本合成的.

随机过程则是一个函数 $\mathbf{x}(\omega, t)$, 将 $\omega \in \Omega$ 和 $t \in T$ 映射到 $\mathbb{R}^d$. 固定住$\omega$, 则 $\mathbf{x}$ 在指标集 $T$ 上给出一条轨迹.

不妨仍然使用丢骰子的例子, 但这次投掷$N$次. 则底层样本空间不妨设置为$\Omega^N$, 最细粒度的 $\mathcal{F}=\mathcal{F}_N$ 就是 $2^{\Omega^N}$. 给定 $\omega\in \Omega^N$ 时, 就给定了全部$N$次的投掷结果.

在我们投掷完第$n\lt N$次时, 得到的结果 $\mathbf{x}(\omega, n)$ 自然是一个随机变量. 然而, 此时 $\mathcal{F}_N$ 真的适合作为第 $n$ 次投掷的事件域吗?

答案是否定的. $\mathcal{F}_N$ 的粒度过细, 在投掷完第$n$次时, 我们尚未知道未来所有投掷的结果. 此时可以讨论的任何事件, 都必定由一组 $\omega$ 合并而成. 这些 $\omega$ 给出的前 $n$ 次投掷结果相同, 而之后才分岔为不同的轨迹.

为此, 我们需要给各$n\in \{1,\dots, N\}$找到它们各自的事件域 $\mathcal{F}_n$, 阻止我们在$n$次投掷时就描述了需要有未来的信息才能描述的事件.

$\mathcal{F}_n$ 依次加细, 即 $\forall s\leq t, \mathcal{F}_s \subseteq \mathcal{F}_t$. $\mathcal{F}_t$ 中多出来的元素, 就是在 $\mathcal{F}_s$ 中所不能描述的事件.
