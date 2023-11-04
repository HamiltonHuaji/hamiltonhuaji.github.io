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

由 bid-ask 价格直接平均所合成的 WAP/mid-price 都是不便之物, 不能足够好地作为合理价格的估计. 例如, 考虑到 bid–ask bounce 效应等微观噪声, mid-price 肯定不是一个鞅. 有必要在以上条件的约束下寻找新的合理价格的估计.

<!-- more -->

寻找资产的合理定价与预测资产价格的未来趋势其实是同一回事. 遵循这一思想, 对 mid-price $M_t$ 变动的任何预测加在当前的 mid-price 上都构成一个更好的估计. 这句话就是说, $$P_t=E[M_t+(M_{\tau_1}-M_t)+(M_{\tau_2}-M_{\tau_1})+\dots \vert F_t]$$
是我们所求的估计之形式. 其中, $F_t$ 表示 $t$ 时刻的所知的任何信息, $M_{\tau_i}$ 表示 mid-price 从 $t$ 时刻起第 $i$ 次变动后的价格.

现在引入一些假设来简化 $E[M_{\tau_{i+1}}-M_{\tau_{i}}\vert F_t]$ 的估计.

1. 只考虑第一档行情的常见指标, 因此 $F_t = (M_t, I_t, S_t)$
2. $(M_t, I_t, S_t)$ 的变动具有马尔可夫性, 且关于$M_t$具有平移对称性.

则最终定义的合理定价为 $$P_\text{micro}^t=\lim_{i\rightarrow\infty}M_t+\sum_{k=1}^i g_k(I_t, S_t)$$

$g_k(I_t, S_t)=E[M_{\tau_{k+1}}-M_{\tau_k}\vert F_t]$ 与通过以下递归方式定义的等价: $$g_{k+1}(I_t, S_t)=E[g_k\vert F_t]$$
