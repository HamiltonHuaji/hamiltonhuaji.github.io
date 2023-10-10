---
title: Notes on Diffusion Models
date: 2023-07-24 15:00:00
mathjax: true
---

以 DDPM 为代表的生成扩散模型在图像生成等领域上具有良好的效果, 并在学AI的文盲们面前小小地展示了一点数学功底的作用.

然而, 笔者读到的几乎所有扩散模型的教程都生硬地打着各种比喻, 提供了远超必须的直观性, 而完全失去了对其后数学动机的介绍. 即, 这样一个乍一看很陌生的过程是如何在保持数学上的有效性时被想到的.

因此, 本文希望展现这一想法可能的一种来龙去脉, 让读者感受到"想到这些关键点, 我也能发现 DDPM".
<!-- more -->

## 问题定义

考虑一个未知的$N$维分布, 其概率密度为$p(x)$[^1]. 我们欲根据这个分布来产生一些新的样本. 分布本身的解析式不得而知, 我们所知道的只有根据这个分布已经取得的一组样本$\lbrace X_i\rbrace$; 因为维度较高且这一分布足够复杂, 即使有解析式也没有什么卵用.

例如, 我们有一些关于猫的图像(每张图像可以用高度乘宽度乘通道数个实数来唯一表达), 它们属于一个$\text{Width}\times\text{Height}\times\text{Channels}$维度的分布.

一个小小的提示是我们的答案将依赖于以下假设: $\lvert\lbrace X_i\rbrace\rvert$, 即样本数量足够大, 因而任何函数在这个分布上的期望都可以用这组样本来取得不错的近似: $$E_{x\sim p(x)}f(x)\simeq \frac{1}{\lvert\lbrace X_i\rbrace\rvert}\sum_i f(X_i)$$ 至于这个近似到底有多好, 在实践中我们是不关心的, 也无力去直接评估; 如果近似得不够好, 那就多加一些样本.

## 从流模型开始

感谢mt19937这类随机数发生器, 我们可以轻松地得到任意区间$[a, b]$[^2]上的均匀分布$\mathcal{U}(\boldsymbol{a}, \boldsymbol{b})$. 这一般是任何实用随机数分布生成算法的起点.

例如, 我们可以首先根据$\mathcal{U}(\boldsymbol{0}, \boldsymbol{I})$采一个样本$z$, 然后将正态分布的CDF的反函数作用在上面, 得到 $$x=\text{CDF}_{\mathcal{N}(\boldsymbol{0},\boldsymbol{I})}^{-1}(z)$$
则这样取得的$x$的系综将遵从正态分布$\mathcal{N}(\boldsymbol{0},\boldsymbol{I})$.[^3]

我们的问题不过是寻找一个高级的随机数发生器, 来根据$p(x)$生成随机样本. 遵照这一想法, 我们希望找到一个函数$f^{-1}(z)$, 将从简单分布$h(z)$中采到的样本$z$映射为我们所需的样本$x$. 这就要求$$p(x)=h(f(x))\cdot\lvert\text{det}(\frac{\partial f}{\partial x})\rvert$$

![$h$中的样本被特定的映射$f^{-1}$转换为遵从$p$分布的样本. 反之, $f$则可以将$p$分布的样本映射回$h$分布](intro.png)

现代神经网络一般被看作一个参数化的函数$f_\theta(x)$, $\theta$为可学习的参数. 随着$\theta$的各种取值, $f_\theta(x)$可以很好地拟合相当广范围内的函数. 由于$p(x)$没有可用且足够简单的解析式, 则我们需要从数据中学习出一个合适的变换$f(x)=f_\theta(x)$. 为了驱使神经网络满足上面的目标, 我们可以将左右两边分布的任意散度(例如, Kullback–Leibler divergence)作为优化目标 [^4] : $$\begin{aligned}\text{argmin}_\theta\ L&=-E_{x\sim p(x)} \log\frac{h(f_\theta(x))\cdot\lvert\text{det}(\frac{\partial f}{\partial x})\rvert}{p(x)}\\&=-E_{x\sim p(x)} \log h(f_\theta(x))\cdot\lvert\text{det}(\frac{\partial f}{\partial x})\rvert+\text{Const}\end{aligned}$$
此处利用了KL散度$\text{KL}(p\vert\vert q)\propto E_{x\sim p(x)}\log\dfrac{q(x)}{p(x)}$的事实. 当这一散度足够小时, 两个分布的差异就足够小, 因而上面的目标就满足得足够好. 容易看出, $f_\theta(x)$的形式需要满足

+ 容易求逆
+ 雅可比行列式$\text{det}(\frac{\partial f}{\partial x})$便于计算(例如, 恒定为$1$)
+ 具有足够的表达力, 以捕捉复杂分布的映射关系

容易想象, 前面这两点往往要求$f_\theta(x)$具有简单的形式, 而这又限制了函数的表达力. 以上的样本生成框架就被称为流模型.

以[NICE: Non-linear Independent Components Estimation](https://arxiv.org/abs/1410.8516)为典型的许多工作都较好地实现了上面三项要求. 它们往往使用多个堆叠的简单变换来表示$$f_\theta(x)=(f_1\circ f_2 \circ \dots)(x)$$

![流模型中, $h$分布被多个堆叠的映射逐步转换为$p$分布](flow.png)

其中, $f_i(x)$的雅可比矩阵的行列式均为$1$, 且它们的逆都可以简单地求出.

{% spoiler NICE的实现细节 %}
设$x=[x_1, x_2]$是$f_i$的输入, 可以被视为$x_1,x_2$的简单拼接.
则令 $$\begin{align}h_1&=x_1\\h_2&=x_2+m(x_1)\\h&=[h_1, h_2]=f_i(x)\end{align}$$ 为$f_i$的输出, $m$为任意函数.

容易看出, 在给定$h$时, $x_1,x_2$均可被简单地还原出来.

而 $$\frac{\partial f_i}{\partial x}=\left[
\begin{array}{ccccc}
1 & 0 & \dots & 0 & 0 \\
0 & 1 & \dots & 0 & 0 \\
\dots & \dots & \dots & \dots & \dots \\
? & ? & \dots & 1 & 0 \\
? & ? & \dots & 0 & 1 \\
\end{array}
\right]$$

因此, $\text{det}(\frac{\partial f_i}{\partial x})=1$.
{% endspoiler %}

## ODE描述的流模型

不难想到, NICE中使用的函数, 其堆叠层数越多, 所具有的非线性拟合能力就越强. 然而另一方面, 每个被堆叠的层可选取的函数形式都太受限了, 也带有过多的人工设计的痕迹, 因而整个堆叠所能等价的函数只是函数空间中的一小部分.

一个自然的想法是, 取堆叠层数$N$无穷大的极限. 在这一极限下, 每一层均应趋近于一个恒等变换加上与$\frac{1}{N}$同阶无穷小的变换, 以保持整个变换存在极限.

![当流模型的堆叠层数趋近无穷时, 样本所受的变换可以看作随时间连续演化](infty.png)

稍有物理背景的读者可能会看出, 对一个量连续施加上面所说的无穷小变换, 可以看作这个量在随着时间不停地演化.

$$\begin{align}x \rightarrow \dots \rightarrow x_t \rightarrow x_{t+\Delta t} \rightarrow \dots \rightarrow z\end{align}$$

因此我们在前述极限下, 将层数$l$看作正比于"时间"$t$的量. 一个系综中的样本每经过一层的变换, 原有的分布$p_t$就被变换为一个新的分布$p_{t+\Delta t}$. 在这一想法的提示下, 我们希望改用关于时间变量$t$的ODE形式$$dx=f_t(x)dt$$来取代流模型, 建立起两个概率密度$p_0$和$p_T$描述的分布之间的映射; 我们希望根据$p_0$取得的样本构成的系综在ODE描述的演化下具有$p_T$的概率密度. 该ODE的逆过程, 则能将遵从$p_T$分布的系综变换回$p_0$. 这一逆过程的存在是非常自然的.

此时, 我们实现了前面的要求. 这一过程对于$p_t$具有一个自然的连续性约束, 即$p_t(x_t)dx_t$在ODE过程中是守恒的. 参考可压缩流体的连续性方程, 我们可以写出$$\frac{\partial p_t(x)}{\partial t}+\nabla\cdot [p_t(x)f_t(x)]=0$$

<!-- 值得一提的是第二条要求的满足, 这有赖于物理学中的[刘维尔定理](https://zh.wikipedia.org/zh-sg/%E5%88%98%E7%BB%B4%E5%B0%94%E5%AE%9A%E7%90%86_(%E5%93%88%E5%AF%86%E9%A1%BF%E5%8A%9B%E5%AD%A6)), 即相空间的分布函数沿着系统的轨迹是常数; 这在我们的问题中, 即意味着$\forall t, p_0(x_0)=p_t(x_t)=p_T(x_T)$, 其中$x_t$是某一个样本点在$t$时刻演化成的对象, 因此这一演化所代表的变换的雅可比矩阵的行列式自然恒为$1$.  -->

这一问题有非常清晰的物理意义, 即将$t$看作原分布之外的一个额外维度, 一组粒子(样本点)在$t=0$的起始面上沿着一个无散向量场的场线运动, 落在$t=T$的终止面上构成了采样的一些样本. 由于粒子不会消失, 也不会在向量场中随时间累积, 因此连续性约束是非常自然的.

## SDE描述的流模型

在上文中, 我们设想了使用ODE来代替人工设计的可逆变换, 搭建源分布和目标分布之间的桥梁. 然而ODE并不是最广的那一类变换的描述方法. 与本文第一张图片类似, 在ODE中, 一组样本在时间中的轨迹构成了"流管"的边界, "流管"中的样本不会流动到"流管"外面去.

然而, 这一性质在我们的目的中其实并不需要满足. 流管内外的样本完全可以随机交换, 只要从统计上说流管的流量是守恒的即可. 换句话说, 前面的ODE只需要负责描述概率密度的"流动", 而无需对单个样本的采样过程负责. 因此前述的ODE可以进一步扩展为SDE来描述具体样本的采样过程, 加上一个随机项, 写为$$dx_t=f_t(x_t)dt+g_t\varepsilon\sqrt{dt},\ \varepsilon\sim\mathcal{N}(\boldsymbol{0},\boldsymbol{I})$$

### Fokker-Planck 方程

在给定单样本演化所遵从的以上SDE时, 我们也可以写出其概率密度所要遵从的方程, 即所谓 Fokker-Planck 方程:

{% spoiler Fokker-Planck 方程的推导细节 %}
为了推导 F-P 方程, 我们可以先写出它的传播子. 即, 若$t$时刻, $p_t(x)=\delta(x-x_t)$, $t':=t+\Delta t$时刻$x_{t'}$遵从的概率密度函数$p_{t'}(x)$.

由于 $$\begin{align}P(X_{t+\Delta t}\leq x_{t+\Delta t})&=\iint_{x+f_t(x)\Delta t+g_t\varepsilon\sqrt{\Delta t}\leq x_{t+\Delta t}} p_t(x)p(\varepsilon) d x d\varepsilon\\&=\iint_{x+f_t(x)\Delta t+g_t\varepsilon\sqrt{\Delta t}\leq x_{t+\Delta t}} \delta(x-x_t)p(\varepsilon) d x d\varepsilon\\&=\int_{x} \delta(x-x_t)[\int_{g_t\varepsilon\sqrt{\Delta t}\leq x_{t+\Delta t}-x-f_t(x_t)\Delta t}p(\varepsilon) d\varepsilon] dx\\&=\int_{\varepsilon \leq \frac{1}{g_t\sqrt{\Delta t}}(x_{t+\Delta t}-x_t-f_t(x_t)\Delta t)}p(\varepsilon)d\varepsilon\end{align}$$

可以知道 $$p_{t'}(x_{t'})=\frac{1}{g_t\sqrt{2\pi\Delta t}}\exp\lbrace-\frac{(x_{t'}-x_t-f_t(x_t)\Delta t)^2}{2g_t^2\Delta t}\rbrace:=G(x_{t'}, t'\vert x_{t}, t)$$

因此在任意给定的$t$时刻概率密度$p_t(x)$下, $t+\Delta t$时刻$x_{t+\Delta t}$遵从的概率密度函数$p_{t+\Delta t}(x)$可以使用传播子与$p_t(x)$的卷积得到: $$p_{t'}(x_{t'})=\int G(x_{t'}, t'\vert x_{t}, t) p_t(x_t) d x_t$$

以上表达式都是在$\Delta t\rightarrow 0$的极限下而言的. F-P方程是将概率密度函数对时间的偏导和一些量对空间的偏导联系起来的方程式, 因此我们应当首先求 $\dfrac{\partial p_{t'}(x_{t'})}{\partial t'}$, 然后看能凑出何种形式.

$$\begin{align}
\frac{\partial p_{t'}(x_{t'})}{\partial t'}
&=\int\frac{\partial G(x_{t'}, t'\vert x_{t}, t)}{\partial t'}  p_t(x_t) d x_t
\end{align}$$

而
$$\begin{align}
\frac{\partial G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}}
&=-\frac{x_{t'}-x_t-f_t(x_t)\Delta t}{g_t^2\Delta t} G(x_{t'}, t'\vert x_{t}, t)
\end{align}$$
$$\begin{align}
\frac{\partial^2 G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}^2}
&=\frac{(x_{t'} - x_t) (x_{t'} - x_t - 2 f_t(x_t) \Delta t) + \Delta t (-gt^2 + f_t(x_t)^2 \Delta t)}{g_t^4\Delta t^2} G(x_{t'}, t'\vert x_{t}, t)
\end{align}$$

因此可以凑出一个关于$G$的等式
$$\begin{align}
\frac{\partial G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}}=\frac{g_t^2}{2}\frac{\partial^2 G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}^2}-f_t(x_t)\frac{\partial G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}}
\end{align}$$

用此式取代掉上面的积分中$G$对$t'$的偏导, 并取$\Delta t\rightarrow 0$的极限. 考虑到 $$\lim_{\Delta t\rightarrow 0} G(x_{t+\Delta t}, t+\Delta t\vert x_{t}, t)=\delta(x_{t+\Delta t}-x_t)$$ 利用$\delta$函数的性质, 有
<!-- 
$$\begin{align}
\lim_{\Delta t\rightarrow 0} \frac{\partial p_{t'}(x_{t'})}{\partial t'}
&=\lim_{\Delta t\rightarrow 0} \int \lbrace \frac{g_t^2}{2}\frac{\partial^2 G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}^2}-f_t(x_t)\frac{\partial G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}} \rbrace p_t(x_t) d x_t\\
\end{align}$$ -->

$$\begin{align}
\lim_{\Delta t\rightarrow 0} \frac{\partial p_{t'}(x_{t'})}{\partial t'}
&=\lim_{\Delta t\rightarrow 0} \int \lbrace \frac{g_t^2}{2}\frac{\partial^2 G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}^2}-f_t(x_t)\frac{\partial G(x_{t'}, t'\vert x_{t}, t)}{\partial x_{t'}} \rbrace p_t(x_t) d x_t\\
&=\int \lbrace \frac{g_t^2}{2}\frac{\partial^2 p_t(x_t)}{\partial x_{t}^2}-\frac{\partial f_t(x_t)p_t(x_t)}{\partial x_{t}} \rbrace \delta(x_{t'}- x_{t}) d x_t\\
&=\frac{g_t^2}{2}\frac{\partial^2 p_{t'}(x_{t'})}{\partial x_{t'}^2}-\frac{\partial f_t(x_t)p_{t'}(x_{t'})}{\partial x_{t'}}
\end{align}$$

或者, 更一般地, 对于高维情况, 有
{% endspoiler %}
$$\frac{\partial}{\partial t}p_t(\boldsymbol{x})=-\nabla\cdot[f_t(\boldsymbol{x})p_t(\boldsymbol{x})]+\frac{g_t^2}{2}\nabla^2p_t(\boldsymbol{x})$$

可以看出, 上式右边的第一项表示在 $f_t$ 这个"流"的驱动下, 因为其散度不为 $0$ 导致的密度累积; 右边第二项则描述了随机运动导致的密度平滑(如果一点处的概率密度比相邻点更低, 则相邻点的概率密度会来填补这个"谷").

### 概率流ODE

事实上, 对于同一个具体的F-P方程, 有一族SDE可以与它对应; 同时还有一个ODE(称作概率流 ODE)给出的概率密度对时间的偏导数与之相同; 这一ODE可以看作前面的SDE族中$g_t$项趋于$0$的结果.

例如, 对于任意$\sigma_t$, 如果在我们考虑的时间范围内, 恒有 $\sigma_t^2\leq g_t^2$, 则可以将F-P方程重写为 $$\begin{aligned}
\frac{\partial}{\partial t}p_t(\boldsymbol{x})
&=-\nabla\cdot[f_t(\boldsymbol{x})p_t(\boldsymbol{x})]+\frac{g_t^2}{2}\nabla^2p_t(\boldsymbol{x})\\
&=-\nabla\cdot[f_t(\boldsymbol{x})p_t(\boldsymbol{x})-\frac{1}{2}(g_t^2-\sigma_t^2)\nabla p_t(\boldsymbol{x})]+\frac{\sigma_t^2}{2}\nabla^2p_t(\boldsymbol{x})\\
&=-\nabla\cdot\left\{\left[f_t(\boldsymbol{x})-\frac{1}{2}(g_t^2-\sigma_t^2)\nabla\log p_t(\boldsymbol{x})\right]p_t(\boldsymbol{x})\right\}+\frac{\sigma_t^2}{2}\nabla^2p_t(\boldsymbol{x})\\
\end{aligned}$$

因此(在已知原有的概率密度$p_t$时)$\sigma_t$诱导了一个新的SDE, 与原SDE具有相同的F-P方程, 因此也具有相同的概率密度演化方式: $$d\boldsymbol{x_t}=\left[f_t(\boldsymbol{x})-\frac{1}{2}(g_t^2-\sigma_t^2)\nabla\log p_t(\boldsymbol{x})\right]dt+\sigma_t\varepsilon\sqrt{dt},\ \varepsilon\sim\mathcal{N}(\boldsymbol{0},\boldsymbol{I})$$

在$\sigma_t=0$时得到的概率流ODE形式是显而易见的: $$d\boldsymbol{x}=\left[f_t(\boldsymbol{x})-\frac{1}{2}g_t^2\nabla\log p_t(\boldsymbol{x})\right]dt$$
<!-- 不难验证, 由概率流ODE导出的概率密度的随动导数是$0$, 这意味着这一ODE演化等价的变换的雅可比行列式为$1$, 从而无需在训练中考虑. -->

{% spoiler 概率流ODE中概率密度的随动导数的计算 %}
$$\begin{aligned}
\frac{Dp_t(\boldsymbol{x_t})}{Dt}
&=\frac{\partial p_t(x)}{\partial t}\vert_{x=\boldsymbol{x_t}}+\frac{d\boldsymbol{x_t}}{dt}\cdot\nabla p_t(\boldsymbol{x_t})\\
&=-\nabla\cdot\left\{\left[f_t(\boldsymbol{x_t})-\frac{1}{2}g_t^2\nabla\log p_t(\boldsymbol{x_t})\right]p_t(\boldsymbol{x_t})\right\}+\left(f_t(\boldsymbol{x_t})-\frac{1}{2}g_t^2\nabla\log p_t(\boldsymbol{x_t})\right)\cdot\nabla p_t(\boldsymbol{x_t})\\
&=-\nabla\cdot\left[f_t(\boldsymbol{x_t})-\frac{1}{2}g_t^2\nabla\log p_t(\boldsymbol{x_t})\right]p_t(\boldsymbol{x_t})
\end{aligned}$$
{% endspoiler %}

### SDE的逆过程

SDE和ODE有一个显著的不同. 在时间反向流动时, ODE描述的过程可以完美地变成其逆过程: $$dx=f_t(x_t)dt, dt\lt 0$$

而从一组遵从$p_T$分布的样本出发, 经由仅将$dt$反号所得的SDE演化, 则一般不能得到$p_0$分布的样本. 为了取得有意义的结果, 我们需要找到另一个SDE, 使得根据$p_T$分布取得的样本演化后能遵从$p_0$分布. 这一问题可以利用概率流ODE得到解决, 即首先找到概率流ODE, 然后利用这一ODE的时间对称性得到逆过程的概率流ODE, 再根据逆过程的概率流ODE得到合适的逆过程SDE族.

具体来说, 考虑 $$d\boldsymbol{x}=\left[f_\tau(\boldsymbol{x})-g_\tau^2\nabla\log p_\tau(\boldsymbol{x})\right]d\tau+g_\tau\varepsilon\sqrt{\lvert d\tau\rvert},\ \varepsilon\sim\mathcal{N}(\boldsymbol{0},\boldsymbol{I}),\ d\tau\lt 0$$

不难发现, 其对应的概率流ODE正好是 $$d\boldsymbol{x}=\left[f_\tau(\boldsymbol{x})-g_\tau^2\nabla\log p_\tau(\boldsymbol{x})\right]d\tau-\frac{1}{2}g_\tau^2\nabla\log p_\tau(\boldsymbol{x})\lvert d\tau\rvert$$

因此可以知道, 这一过程正是SDE的逆过程.

## ODE/SDE流模型的实现

至此我们已经讨论了流模型的无限层极限, 并使用ODE/SDE刻画了这一极限.

回到我们待解决的问题中来. 我们面前有一些从$N$维未知分布$p_0(x)$中采集的样本, 现在我们希望写出/学习到一个ODE/SDE, 描述$p_0(x)$到$p_T(x)\sim \mathcal{N}(\boldsymbol{0},\boldsymbol{I})$(或其他简单分布)之间的变换, 这个变换称为正向过程; 随后, 我们希望能得到这个ODE/SDE的逆过程, 以按照$p_0(x)$来采出新的样本.

对于前一步, 我们可以比较轻易地写出ODE/SDE来实现这一变换.

### ODE流模型示例: PFGM

对于ODE而言, [Poisson Flow Generative Models](https://arxiv.org/abs/2209.11178)就是一个良好的例子. 考虑在样本所在的空间上再加一维$z$. 将所有样本点$x$放置在$[x, 0]$的位置上, 并将其看作是$N+1$维中的电荷, 发射出电场线直到无穷远处. 则这一空间中的电场就是一个完美的无散无旋场. 考虑到任何有限大小的电荷分布在足够远处都等效于一个点电荷, 足够远处的电场线密度必然是球对称而均匀的; 只需在无穷远的球壳上均匀采样一个样本点, 逆着电场线的方向运动到$z=0$处就可以得到$p_0$中的一个样本; 这一运动即为所需的ODE. 为了加速场的计算, 可令神经网络直接学习所得的向量场, 即学习ODE中$f_t(x_t)$项本身.

![PFGM的二维示例. 二维分布被看作三维空间中的电荷, 构建了三维球壳这个二维流形上的均匀分布和原分布的映射关系](PFGM.png)

### SDE流模型示例: DDPM

对于SDE而言, 只需要让样本的演化等价于$p_0$和$p_T$逐步"混合", 就可以得到满足要求的变换. 例如, 令 $$x_{t+\Delta t}=\sqrt{1-\beta_t^2\Delta t} x_t + \beta_t\varepsilon \sqrt{\Delta t},\ \varepsilon\sim\mathcal{N}(\boldsymbol{0},\boldsymbol{I})$$ 时, 在$t\rightarrow\infty$时, $x_\infty\sim\mathcal{N}(\boldsymbol{0},\boldsymbol{I})$

此时, 正向过程的SDE形式完全不包含和具体样本相关的信息, 同一个SDE可以保证任何$p_0$被变换到$p_T$, 问题的全部难度在于SDE的"逆过程"的实现. 从上节的结果可以知道, $\nabla\log p_t(\boldsymbol{x})$含有对逆过程至关重要的信息. 因此, 我们可以使用一个神经网络$s_\theta(x_t, t)$来拟合这一项. 为了利用上样本数据, 关于拟合误差的项(例如, 拟合误差的二范数的期望)需要被写成某个东西在$p_0$上的期望的形式来求值:

由于
$$
\begin{aligned}
\nabla\log p_t(x_t)
&=\frac{E_{x_0\sim p_0(x_0)}\nabla p_t(x_t\vert x_0)}{E_{x_0\sim p_0(x_0)}p_t(x_t\vert x_0)}\\
&=\frac{E_{x_0\sim p_0(x_0)}p_t(x_t\vert x_0)\nabla\log p_t(x_t\vert x_0)}{E_{x_0\sim p_0(x_0)}p_t(x_t\vert x_0)}\\
\end{aligned}
$$

因此在
$$
\begin{aligned}
\text{argmin}_\theta \frac{E_{x_0\sim p_0(x_0)}p_t(x_t\vert x_0)\lvert\nabla\log p_t(x_t\vert x_0)-s_\theta(x_t, t)\rvert^2}{E_{x_0\sim p_0(x_0)}p_t(x_t\vert x_0)}\\
\end{aligned}
$$
的条件下, 有
$$
\begin{aligned}
s_\theta(x_t,t)=\nabla\log p_t(x_t)
\end{aligned}
$$

由于上式需要对任意$x_t, t$成立, 故被优化的目标函数应再对$t$与$x_t$积分, 即最终优化的目标为
$$
\begin{aligned}
L_{s_\theta} = E_{t\sim\lambda(t)}E_{x_0\sim p_0(x_0)}E_{x_t\sim p_t(x_t\vert x_0)}\lvert\nabla\log p_t(x_t\vert x_0)-s_\theta(x_t, t)\rvert^2\\
\end{aligned}
$$
这被称为得分匹配(Score Matching)的损失函数.[^5]

在本小节开头给出的SDE形式下, 这一损失函数可以进一步改写. 容易发现, $x_t\vert x_0$可以直接通过在$\alpha_t x_0$上加入一个遵从$\mathcal{N}(\boldsymbol{0},\sigma_t\boldsymbol{I})$的随机项而采样出来, 其中$\sigma_t^2=\sum\beta_t^2\Delta t=\int \beta_t^2 dt$, $\alpha_t=\sqrt{1-\sigma_t^2}$. 此时可以得到
$$
\begin{aligned}
\nabla\log p_t(x_t\vert x_0)
&=\nabla\log \frac{\exp\left\{-\frac{(x_t-\alpha_t x_0)^2}{2\sigma_t^2}\right\}}{\sqrt{2\pi\sigma_t}}\\
&=-\frac{\nabla(x_t-\alpha_t x_0)^2}{2\sigma_t^2}\\
&=-\frac{x_t-\alpha_t x_0}{\sigma_t^2}\\
\end{aligned}
$$

简单进行一下重参数化, 即定义新的待学习函数 $$-\frac{\epsilon_\theta(x_t, t)}{\sigma_t^2}=s_\theta(x_t, t)$$, 则原来的得分匹配损失函数变为
$$
\begin{aligned}
L_{\epsilon_\theta}
&=E_{t\sim\lambda(t)}E_{x_0\sim p_0(x_0)}E_{x_t\sim p_t(x_t\vert x_0)}\lvert (x_t-\alpha_tx_0) -\epsilon_\theta(x_t, t)\rvert^2\\
&=E_{t\sim\lambda(t)}E_{x_0\sim p_0(x_0)}E_{\varepsilon\sim \mathcal{N}(\boldsymbol{0},\boldsymbol{I})}\lvert \varepsilon -\epsilon_\theta(\alpha_tx_0+\sigma_t\varepsilon, t)\rvert^2\\
\end{aligned}
$$
再通过一些tricky的变换[^6], 我们可以再从这一形式得到烂大街的DDPM原理对应的那种损失函数(即让神经网络预测每一个时间步施加在数据上的噪声); 然而本文将止步于此, 原因有三:

+ 预测噪声的形式模糊了SDE描述的流模型这一远为通用的思维框架
+ 目前所得这一形式的损失函数训练效果更好
+ 加噪-降噪的形式暗示我们生成过程是移除噪声的过程; 从这一观点出发无法看到由概率流ODE诱导的一族逆向SDE过程的存在, 更不能看到 Schrödinger Bridge 方法等更广泛的生成模型的世界.

此时, 假设我们已经得到了$s_\theta(x_t, t)$. 我们就有多种选择来采样得到所需的样本. 例如, 可以直接根据前面的逆向SDE形式重新进行一遍采样, 也可以导出概率流ODE进行确定性的生成. 这一过程可以通过已经脍炙人口的各种ODE数值求解方法来加速.

概率流ODE作为一个噪声向量到目标分布的确定性变换还允许我们对噪声向量插值, 以获得两个样本的在分布中的混合结果(例如, 符合逻辑的图片插值, 显然不能直接对同一位置的像素颜色插值, 这样生成的图片不在分布中).

## 扩展

// TODO: 写完DDPM相关就失去了写的兴趣, 有空的时候再回来写吧~

+ [ ] latent diffusion
+ [ ] 条件生成模型
+ [ ] ...

[^1]:本文中直接使用某个分布的概率密度的函数名来称呼该分布.例如,h(x)中的样本可以理解为一组遵从h(x)概率密度采样得到的样本
[^2]:考虑到单个实数的测度为0,我们就不考虑开闭区间的问题了.
[^3]:真正的正态分布发生器不是这样工作的,这只是一个理论上可行的方案,因为这个CDF的逆没有解析式.
[^4]:有的人喜欢说流模型的训练是在做最大似然估计.然而,描述两个分布是否接近有许多种度量,最大化对数似然只是在你使用KL散度时的等价目标;没有任何实现难度之外的理由阻止你使用JS散度或者Wasserstein距离之类的玩意,此时的目标函数自然不等价于对数似然.顺便一提,我很讨厌散度这个名字,因为它令人错误地想到矢量分析,翻译为分歧度也许会更好.
[^5]:见[这里](https://arxiv.org/abs/2011.13456)
[^6]:一个逆向的推导见[这里](https://kexue.fm/archives/9119)
