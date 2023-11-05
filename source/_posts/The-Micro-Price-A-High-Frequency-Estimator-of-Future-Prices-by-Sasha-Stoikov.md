---
title: 'The Micro-Price: A High Frequency Estimator of Future Prices by Sasha Stoikov'
date: 2023-11-04 19:36:35
mathjax: true
tags:
- quant
---

回顾资产定价之常识, 即资产未来(远期)价格折现值之期望为当前的合理价格. 若暂不考虑折现值, 则合理价格作为随机过程应当是一个鞅过程.

1. 合理价格之估计本身也应当是一个鞅过程.
2. 增加所考虑的市场上的信息时, 这一估计应当一致逼近合理价格.
3. 这一估计之更新频率应当能与订单流频率相比拟.

由 bid-ask 价格直接平均所合成的 WAP/mid-price 都是不便之物, 不能足够好地作为合理价格的估计. 例如, WAP/mid-price 都没有其鞅性的理论依据; 前者容易受订单簿的噪声干扰, 后者则完全不考虑订单簿的订单量信息. 有必要在以上三个条件的约束下寻找新的合理价格的估计.

<!-- more -->

寻找资产的合理定价与预测资产价格的未来趋势其实是同一回事. 遵循这一思想, 对 mid-price $M_t$ 变动的任何预测加在当前的 mid-price 上都构成一个更合适的定价. 这句话就是说, $$P_t=E[M_t+(M_{\tau_1}-M_t)+(M_{\tau_2}-M_{\tau_1})+\dots \vert F_t]$$
是我们所求的估计之形式. 其中, $F_t$ 表示 $t$ 时刻的所知的任何信息, $M_{\tau_i}$ 表示 mid-price 从 $t$ 时刻起第 $i$ 次变动后的价格.

现在引入一些假设来具体化 $E[M_{\tau_{i+1}}-M_{\tau_{i}}\vert F_t]$ 的形式.

1. 只考虑第一档行情的常见指标, 因此 $F_t = (M_t, I_t, S_t)$
2. $(M_t, I_t, S_t)$ 的变动具有马尔可夫性, 且关于$M_t$具有平移对称性.

则合理定价的形式为 $$P_\text{micro}^t=\lim_{i\rightarrow\infty}M_t+\sum_{k=1}^i g_k(I_t, S_t)$$

$g_{k+1}(I_t, S_t):=E[M_{\tau_{k+1}}-M_{\tau_k}\vert F_t]$ 与通过以下方式定义的等价:
$$\begin{aligned}
g_1(I_t, S_t)&=E[M_{\tau_1}-M_t\vert F_t]\\
g_{k+1}(I_t, S_t)&=E[g_k(I_{\tau_1}, S_{\tau_1})\vert F_t]
\end{aligned}$$

直观来看, $g_k\rightarrow g_{k+1}$, 即第$\tau_k$次价格变动, 是 $(M_{\tau_k}, I_{\tau_k}, S_{\tau_k})$ 按照马尔科夫链给出的概率演化了一次.

然而, 这一形式之收敛性尚未可知, 其具体数值计算之方式尚有待说明.

// TODO

<!-- // 以下配合原文阅读

$B$矩阵表示$G^k$演化一次的算符. 由于 $$P_\text{micro}^t=M_t+\sum_{k=1}^\infty B^k G^1(I_t, S_t)$$ $$(\lim_{i\rightarrow\infty} B^i)G^1=0$$是其收敛的自然的保证. -->
