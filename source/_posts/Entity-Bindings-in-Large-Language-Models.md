---
title: Entity Bindings in Large Language Models
date: 2023-11-30 14:53:36
mathjax: true
tags:
- Large Language Models
---

本文是关于文章 [How do Language Models Bind Entities in Context?](https://arxiv.org/abs/2310.17191) 的笔记.

Entity Binding, 即实体绑定, 是 LLM 在上下文中将属性与实体联系起来的能力. 例如, 如下情景中, `Alice` 和 `Bob` 分别与 `lives in Paris` 和 `lives in Bangkok` 属性绑定. LLM 根据绑定好的属性进一步回答随后的问题.

```plaintext
Context: Alice lives in the capital city of France. Bob lives in the capital city of Thailand.
Question: Which city does Bob live in?
```

<!-- more -->

{% graphviz "entity binding" %}
digraph {
  rankdir = LR;
  node [shape=record];
  K [label="<K0>K0|<K1>K1|<K2>K2|..."]
  E0 [label="E0:Alice"];
  A0 [label="A0:lives in Paris"];
  A0 -> K:K0 -> E0;

  E1 [label="E1:Bob"];
  A1 [label="A1:lives in Bangkok"];
  A1 -> K:K1 -> E1;
}
{% endgraphviz %}

以往对 LLM 中编码的信息的解读更偏向于基元(primitive)信息. 理解 LLM 中实体绑定的机制, 有助于我们解读 LLM 中的组合(constructive)信息.

{% spoiler 符号定义 %}

考虑由一组 token $\{t_1, t_2, \dots, p, \dots, t_n\}$ 组成的上下文 $\mathbf{c}$. 在 $\mathbf{c}$ 中, 语义上有一组属性$A_0, A_1, \dots$与一组实体$E_0, E_1, \dots$绑定, 记为 $$\mathbf{c}=\{A_0\rightarrow E_0, A_1\rightarrow E_1, \dots\}$$
对于某个 token $p$, LLM 的每一层产生了 $d_\text{model}$ 维的激活向量; 因此对于 $p$ 在 $\mathbf{c}$ 中的表征, LLM 的激活值可以表示为 $$Z_p(\mathbf{c}) \in \mathbb{R}^{n_\text{layers}\times d_\text{model}}$$
而上下文的表征则记为 $$Z(\mathbf{c})=\{Z_p(\mathbf{c})\vert p\in \mathbf{c}\}$$
对于前文 `Alice lives in the capital city of France` 的例子, $E_0=\text{Alice}$, $A_0=\text{the capital city of France}$, $$Z(\mathbf{c})=\{Z_{E_0}, Z_{\dots}, Z_{A_0}, Z_{\dots}, Z_{E_1}, Z_{\dots}, Z_{A_1}\}$$

为了进行实验, 有时需引入与上下文 $\mathbf{c}$ (称作目标上下文)有所不同的 $\mathbf{c'}$ (称作源上下文). 有必要定义在 $Z(\mathbf{c})$ 中进行反事实干预的记号, 即 $$Z(\mathbf{c})/.\{Z_p\rightarrow Z_p'\}:=\{Z_0, Z_1, \dots, Z_{p-1}, Z_p', Z_{p+1}, \dots\}$$ 表示从源上下文 $\mathbf{c'}$ 中取得 token $p'$ 的表征, 并替换掉目标上下文 $\mathbf{c}$ 中 token $p$ 的表征.
{% endspoiler %}

本文找到的绑定机制称作 binding ID. 具体来说, 对于上下文 $c$ 中的第 $k$ 个实体 $e$ 和其属性 $a$, LLM 做了以下三件事:

1. 在 $e$ 所属的 token 的激活值上编码 $$Z_{e}=\Gamma_E(e, k)=f_E(e) + b_E(k)$$
2. 在 $a$ 所属的 token 的激活值上编码 $$Z_{a}=\Gamma_A(a, k)=f_A(e) + b_A(k)$$ $\{b_E(k), b_A(k)\}$ 称作 binding ID. 这两步使得 $e, a$ 各自与 binding ID 联系了起来.
3. 为了回答有关 $e$ 的问题, LLM 从上下文编码 $Z_c$ 中取得与 $e$ 具有相同 $k$ 值的属性 $a$. 即根据 $e$ 求得 $b_E(k)$, 并以此注意到编码了 $b_A(k)$ 的那些 $a$ 对应的 token.

相应于任何假说, 都有必要据此推导出一些预测并加以实验验证. 作者预言的两项特性为

1. 可分解性. 如果将 $Z_{A_k}$ 替换为在其它上下文下取得的 $Z_{A_k'}$, 则可以预期模型将相信 $c/.\{A_k\rightarrow A_k'\}$. 这是因为属性仅仅通过 binding ID 与实体间接联系. 类似的, 如果替换实体的表征, 这一属性也成立.
2. 位置独立性. 如果将同一上下文的 $Z_{A_i}$ 与 $Z_{A_j}$ 替换, 则模型的信念不会发生改变.

实际进行的实验考虑了两对实体与属性. 作者计算了两个上下文 $$\begin{align*}\mathbf{c}&=\{A_0\rightarrow E_0, A_1\rightarrow E_1\}\\\mathbf{c'}&=\{A_0'\rightarrow E_0', A_1'\rightarrow E_1'\}\end{align*}$$ 上的表征, 并使用 causal mediation analysis 交换了两个上下文中的部分表征. 具体来说, 作者考虑了对 $k \in \{0, 1\}$, 替换 $Z_{E_k}\rightarrow Z_{E_k'}$, $Z_{A_k}\rightarrow Z_{A_k'}$ 以及它们的组合, 并收集了模型关于 $\{E_0, E_1, E_0', E_1'\}$ 的查询输出各属性的概率.

{% raw %}
<style>
    .blue {
        background-color: blue;
        color: white;
        margin: 0px;
        line-height: 1.2;
        white-space: pre;
    }
    .green {
        background-color: green;
        color: white;
        margin: 0px;
        line-height: 1.2;
        white-space: pre;
    }
    .grid-container {
        display: grid;
        grid-template-columns: max-content max-content 1fr 1fr 1fr 1fr; /* 第一列宽度为最宽的内容，其余列平分剩余空间 */
        grid-auto-rows: min-content;
        grid-gap: 0px;
        align-items: center;
    }
    .grid-item {
        border: 1px solid black;
        padding: 0px;
        white-space: nowrap;
        line-height: 1.2;
        display: flex;
        align-items: center;
        height: 100%;
    }
    .high {
        background-color: #cc4444;
    }
    .low {
        background-color: #4444cc;
    }
</style>
<div class="grid-container">
    <div class="grid-item">Context</div>
    <div class="grid-item">Query</div>
    <div class="grid-item">P(Paris)</div>
    <div class="grid-item">P(Bangkok)</div>
    <div class="grid-item">P(Rome)</div>
    <div class="grid-item">P(Mumbai)</div>
    <!-- context c -->
    <div class="grid-item">
        <span class="blue">Alice lives in Paris. Bob lives in Bangkok.</span>
    </div>
    <div class="grid-item">
        <span class="blue">Alice lives in ...</span>
    </div>
    <div class="grid-item high"/><div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/>
    <div class="grid-item">
        <span class="blue">Alice lives in Paris. Bob lives in Bangkok.</span>
    </div>
    <div class="grid-item">
        <span class="blue">Bob lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item high"/><div class="grid-item low"/><div class="grid-item low"/>
    <!-- context c' -->
    <div class="grid-item">
        <span class="green">Tracy lives in Rome. John lives in Mumbai.</span>
    </div>
    <div class="grid-item">
        <span class="green">Tracy lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item low"/><div class="grid-item high"/><div class="grid-item low"/>
    <div class="grid-item">
        <span class="green">Tracy lives in Rome. John lives in Mumbai.</span>
    </div>
    <div class="grid-item">
        <span class="green">John lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/><div class="grid-item high"/>
    <!-- replace attribute -->
    <span class="grid-item">
        <span class="blue">Alice lives in </span><span class="green">Rome</span><span class="blue">. Bob lives in Bangkok.</span>
    </span>
    <div class="grid-item">
        <span class="blue">Alice lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item low"/><div class="grid-item high"/><div class="grid-item low"/>
    <div class="grid-item">
        <span class="blue">Alice lives in </span><span class="green">Rome</span><span class="blue">. Bob lives in Bangkok.</span>
    </div>
    <div class="grid-item">
        <span class="blue">Bob lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item high"/><div class="grid-item low"/><div class="grid-item low"/>
    <!-- replace entity -->
    <span class="grid-item">
        <span class="green">Tracy</span><span class="blue"> lives in Paris. Bob lives in Bangkok.</span>
    </span>
    <div class="grid-item">
        <span class="blue">Alice lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/>
    <span class="grid-item">
        <span class="green">Tracy</span><span class="blue"> lives in Paris. Bob lives in Bangkok.</span>
    </span>
    <div class="grid-item">
        <span class="blue">Bob lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item high"/><div class="grid-item low"/><div class="grid-item low"/>
    <span class="grid-item">
        <span class="green">Tracy</span><span class="blue"> lives in Paris. Bob lives in Bangkok.</span>
    </span>
    <div class="grid-item">
        <span class="green">Tracy</span><span class="blue"> lives in ...</span>
    </div>
    <div class="grid-item high"/><div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/>
    <span class="grid-item">
        <span class="green">Tracy</span><span class="blue"> lives in Paris. Bob lives in Bangkok.</span>
    </span>
    <div class="grid-item">
        <span class="green">John</span><span class="blue"> lives in ...</span>
    </div>
    <div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/><div class="grid-item low"/>
</div>
{% endraw %}

如上表所示, 模型给出的概率(红色为高)与可分解性给出的预测相符. 直接将蓝色上下文中的 `Paris` 的表征替换为根据绿色上下文求出的 `Rome` 的表征导致模型相信 `Alice` 住在罗马.

另一方面, 考虑到 Transformer 中, 模型对于 token 位置的感知完全基于 positional embedding, 因此也只需调整此项来"挪动" token 的位置. 这一挪动记作 $\{X_k\rightarrow k'\}$

作者考虑了将 $E_0$ 的位置逐渐与 $E_1$ 的位置交换的挪动方式: $$\{X_{E_0}\rightarrow x, X_{E_1}\rightarrow X_{E_1}-(x-X_{E_0})\}, \forall x \in \{X_{E_0}, X_{E_0}+1, \dots, X_{E_1}\}$$

在这一过程中, 可以绘制出模型预测各有关 token 的对数概率与$x$的关系. 作者发现, 尽管这些概率与 $x$ 有关, 最终的结论仍然成立, 因为 $x$ 改变时, 各概率同步变化, 不论某个属性是否被绑定了, 说明这一变化只是模型学到的其它 bias.

以上实验只证明了实体绑定的过程中, 存在一个中间变量; **只要实体和属性分别绑定到同一个中间变量值上, 这一对实体和属性就实现了绑定**. 然而, 这些实验尚不能确定中间变量值的形式(即 $\Gamma_E(e, k), \Gamma_A(a, k)$ 的形式). 不妨在此继续贯彻先猜想后验证的优良传统, 依据 LLM 大概率使用线性的信息编码这一先验信息, 猜测 $\Gamma_E(e, k), \Gamma_A(a, k)$ 这些双参映射是由关于两个参数的独立部分相加而成的.

在这一假设下, $$\begin{align*}\Delta_A(k)&:=\Gamma_A(\alpha, k)-\Gamma_A(\alpha, 0)\quad\forall \alpha\\\Delta_E(k)&:=\Gamma_E(\beta, k)-\Gamma_E(\beta, 0)\quad\forall \beta\end{align*}$$ 是与 $\alpha, \beta$ 无关的值(实际上可能有弱相关). 这一值的实际估计可以通过对数据集采样求平均来得到, 以去除该弱相关性.

在获得了 $\Delta_A(k)$ 与 $\Delta_E(k)$ 后, 我们可以进一步地干涉实体绑定过程中的这个中间变量. 我们预期, $\{Z_{A_0}\rightarrow Z_{A_0}+\Delta_A(1), Z_{A_1}\rightarrow Z_{A_1}-\Delta_A(1)\}$ 这一干涉可以使得模型认为 $A_0, A_1$ 发生了交换, 将 $A_0, E_1$ 和 $A_1, E_0$ 进行绑定.
