---
title: Two Approaches to LLM Interpretability
date: 2023-10-10 14:48:37
mathjax: true
tags:
- Large Language Models
---

大语言模型因其卓越的性能吸引了大量的关注. 人们自然而然地会想问:

+ 它们*内部*是如何工作的(*internal* mechanics)
+ 我如何能让它们变得更好

通过回答第一个问题, 也许可以帮助我们回答第二个问题

<!-- more -->

## What does the term *Internal* Mean?

唯象学(Phenomenology)关注的是总结现象之间的规律. 而这是大部分 LLM 研究在做的事情. 例如, `LIMA: Less Is More for Alignment` 说的是当你的数据质量足够高, 1000条数据就足以得到还不错的指令微调效果. But, *Why*?

{% spoiler Example: Two Kinds of Interpretation %}

给 LLM 输入一条指令

```plaintext
Translate this sentence to English: 今天做核酸的队长死了, 所以我等了好久
```

目前的几乎每一个大模型(GPT3.5, GPT4, 文心一言, ...)都会输出与
```plaintext
Today, the team leader responsible for conducting nucleic acid tests died, so I had to wait for a long time.
(今天/做核酸的队长/死了, 所以我等了好久 vs 今天/做核酸的队/长死了, 所以我等了好久)
```
类似的句子(而且即使使用CoT之类的提示技术也不能改正这一点).

对于这个例子, 我们可以给出几种解释(这一现象我目前还没找到真正的解释, 仅作示意):

+ LLM输出这些是因为没有grounding, 即不具有文本与真实世界之间关联的知识.
+ LLM输出这些是因为没有常识, 即不知道排的队比较长就需要等很久/做核酸测试没有队长.
+ 训练数据里不包含针对性的歧义句.
+ 上下文让LLM内部对"长"这个token的表征同时包含相应于"队长"和"长度"的激活值, 但后者的影响被前者掩盖了; 通过在上文中插入一些"长"作"长度"义的句子, 可以增强后者, 让模型进行正确的翻译.

这四种解释都引入了某些因果关系, 即*干涉*变量A就会决定变量B的关系.

然而前三种解释都属于唯象学的范畴, 停留在"数据能教会LLM一些东西"的层面: 如果LLM没有学会某些知识, 任务就不能很好地完成.(Q: 什么是学会知识?) 而基于它们导出的任何改善模型的方法都只能服从修改数据-训练的范式.

第四种则不满足于此. 这种解释要求我们打开模型的黑盒子, 将训练数据与现象的因果关系*还原*成模型响应与其具体权重/具体表征之间的因果关系. 这就要求我们开发可解释性方法对模型内部进行检视或干涉, 以获得新的认识; 并将这些认识用于预测和定制模型的行为, 或激发模型的能力.

{% endspoiler %}

可解释性相关的研究则试图将 LLM 的行为还原到更基本的单元, 来创建"cognitive science of AI"; 这些单元可以是 LLM 的内部状态的子集, 也可以是由 LLM 的内部状态合成得出的虚拟状态(例如一些内部状态经过了某个线性/非线性变换的结果).

## The Bottom-Up Approach: Mechanistic Interpretability(MI)

认知科学中的 Sherringtonian view: 认知是节点之间的连接产生的; 这在物质上又是通过大脑中神经元组成的回路来实现的.

众所周知, LLM 对于下一个 token 的预测是通过给出每个 token 相应的概率来实现的; 而这一概率又是最后一个隐藏层的输出经过反嵌入矩阵(unembed matrix)乘法后的输出(logits)进行softmax操作得到的.

基于这一观点, 机械可解释性着力于挖掘 LLM 中的回路, 并将logits视作多个回路的输出的向量和. 模型总的输出则是各回路互相竞争的结果. 所谓"模型学会了知识", 就是指模型中具有这个知识相应的回路, 使得模型在见到合适的模式时预测出这条知识所对应的下一个token.

{% spoiler Example: Induction Heads %}

假设 `D/urs/leys` 是模型在预训练中可能没有见过的单词. 所谓 induction head 的回路, 可以使得模型在上文出现 `Dursleys` 后正确重复这个单词. 换句话说, 这个回路帮助 LLM 产生了最基础的 in-context learning 的能力. 进一步寻找针对更抽象模式的 induction head, 将有助于理解 in-context learning 的涌现等现象.

![induction head 的示意图. 在第1层中的某个注意力头中, `ters` 注意到自己的前一个token `Pot`, 并将该token的值复制到下一层作为key, 自己的值依旧作为value; 第2层的某个注意力头中, 后一个 `Pot` token以自己的值作为 query , 去找 key 相似的 token, 于是注意到了前一个 `ters` token; 则两层注意力头得到的等效注意力, 从第二个 `Pot` 指向第一个 `Pot` 的下一个token `ters`. 模型只需要简单地将 `ters` 输出, 就能根据上文正确地补全 `Potter` 这个单词](induction_head_expanded.png)

![`urs`token的等效注意力指向了前一次`urs`出现时的下一个token`leys`. 因此这导致模型预测`leys`出现在`urs`后面的概率增加了](induction_head_attention.png)

{% endspoiler %}

限于篇幅关系, 不在此处给出更多具体回路的例子. 总结地来说, 目前机械可解释性的研究可以看作逆向工程, 试图捕捉 LLM 进行的计算的重要部分, 并转换为人类可以理解的、具有语义的、可拆分和组合的形式.

这些研究的确可以提供一些有意义的看法. 例如, 通过分析这些回路, 我们可以知道模型在哪些情况下具有泛化能力, 哪些情况下只是记住了训练集. 所谓[逆转诅咒](https://arxiv.org/abs/2309.12288)(Reversal Curse)现象声称, LLM 的训练集中如果只包含 "Beijing is the capital of China", 则 LLM 一般不能自动地泛化得出 "The capital of China is Beijing". 在机械可解释性的眼光下, 学到 "A is B" 只是在 MLP 层中构造了一个查找表, MLP("A is")="B"; 这一查找表自然不会是天然就可逆的(除非在训练集中也见过逆关系), 也不会让 LLM 自动学会 "B is A".

在提供唯象学以外的认识之外, 机械可解释性方法也有其内在的局限.(Q: 我们能通过阅读JVM的源码来预测任何一个java程序的行为吗?)

+ 回路的挖掘和解释往往严重依赖于人工, 自动化的方法还不足以提供很好的结果.
+ 回路挖掘的结果只能用于解释较简单的行为.
+ 神经网络原理上并不能使用回路来*完全*解释其行为. 例如, ResNet在移除一整层的情况下可以是鲁棒的, 这自然超出了依赖多个层精密协作的回路的能力(将这样的网络视作 Neural ODE 会更合适).

## The Top-Down Approach: Representation Engineering(RepE)

认知科学中相反的 Hopfieldian view: 认知是表征空间的结果, 在物质上说, 神经元的活动情况就是表征.

相比于 MI, 这一观点并不关心 LLM 中的输入到表征的映射是如何实现的, 也不关心表征到模型行为的影响是如何产生的; 相反, 它关心表征本身的语义以及干涉表征对模型行为造成的影响本身. 例如, Burns等人（2022）在 LLM 中确定了真实性(truthfulness)的表征, 证明在不少情况下, 即使输出错误信息, 模型也知道正确答案. 相关的发现无疑可以用于直接提升模型的准确率, 而无需构建新的数据集或对模型重新训练.

这一观点可以类比为物理学中的简正模: 一组耦合的谐振子无疑是难以分析的, 但可以使用它们的坐标的线性组合来合成一组新的变量(简正模, 模 for 模式); 新的变量可以具有互相独立(因而更加简单)的运动规律. 而单个谐振子的运动同样可以被视作各模式单独存在时的运动情况的叠加.

在自顶向下的可解释性框架上, 可以进行多种层级的实验.

1. 挖掘相关性的实验. 以 LAT 为代表的方法(见下文)帮助建立了表征与 LLM 行为的相关性. 然而, 表征与 LLM 行为的*因果关系*尚有待进一步的实验评估.
2. 操纵 LLM 的实验. 回顾对于因果的定义, 只有主动干涉因果变量, 才能将相关性上升为因果性.
3. 消融实验. 如果某表征对 LLM 行为是*必要的*, 消除之将导致 LLM 的退化, 才能说该表征与 LLM 行为具有完整的因果关系.
4. 恢复实验. 如果某概念被完全移除, 而同时引入对应的表征, 则可以预计 LLM 将重新具有该概念对应的行为. 这就能说明该表征对于 LLM 的特定行为是充分的.

RepE, 即 [Representation Engineering: A Top-Down Approach to AI Transparancy](https://arxiv.org/abs/2310.01405) 总结了这一方向上的工作. 相应于实验层级 1, 该工作提供了 baseline 方法, 称为 Linear Artificial Tomography(LAT).

{% spoiler Linear Artificial Tomography %}

该方法分为三个主要的步骤. 第一步是选择合适的提示, 激发模型对某个概念的理解或对某个函数的计算. 这被视为神经科学中的"刺激", 以期触发模型内部的响应. 例如, 一个可能的提示模板如下:

```plaintext
Consider the amount of <concept> in the following:
<stimulus>
The amount of <concept> is <?>
```

为了激发模型与遵循指令相关的响应, 如下的提示模板被使用. 其中 `experimental prompt` 使得模型必须执行选定的计算; `reference prompt` 则无需模型进行相关的计算.

```plaintext
USER: <instruction> <experimental/reference prompt>
ASSISTANT: <?>
```

下文中, 针对概念$c$和刺激$s_i$的提示记为$T_c(s_i)$, 针对`experimental/reference prompt`所对应的计算记为$T_f^+$和$T_f^-$.

第二步是收集模型相应的内部状态. 针对基于 Transformer 的 LLM, 有多个内部状态的子集可供选用. 考虑有关概念 `happiness` 的如下提示

```plaintext
The amount of happiness in the scenario is <?>
```

一个自然的选择是收集相应于该概念的 token `happiness` 的表示(即第一层到最后一层之间所有的 Transformer Block 的输出, 并将其拼接成单个向量). 另一个选择则需要考虑到 Decoder-Only LLM 的架构特性. 以上提示中的 token `is` 要求模型立即进行计算并给出答案; 单向注意力导致只有 `is` token 能收集到上下文的全部信息. 因此触发 LLM 计算的最后一个 token 的表示也可以作为模型内部状态的有代表性的子集.

形式上说, 对于概念$c$, 模型$M$, 一组刺激$\{s_i \in S\}$, 这样收集到的内部状态集合可表示为 $$A_c(S) = \{M(T_c(s_i))[-1] \vert s_i \in S\}$$ 对于计算$T_f^\pm$和相应的指令-响应对$(q_i, a_i)\in S$, 内部状态集合为 $$A_f^\pm(S)=\{M(T_f^\pm(q_i, a_i))[-1] \vert (q_i, a_i) \in S\}$$

第三步是构建以前述内部状态为输入的模型, 以满足各种实用目的.

这一模型可以是在标签数据集上训练的线性模型. 例如, 为了检验模型是否能仅从文本预训练中学习到对世界地理位置的认知(即模型内部是否建模了所谓世界模型的一部分), 可以构造一个包含各种地名的概念集合$C=\{天安门广场, 旧金山, 自由女神像, 凯旋门, 埃及, \dots\}$和针对每个概念$\{c\in C\}$构造的各类刺激$S_c$, 并以$\{A_c(S_c)\vert c\in C\}$作为输入来预测这些概念的经纬度$L_c$. 如果这样的简单线性模型能准确地预测出$L_c$, 就说明模型内部确实编码了这些概念的地理位置信息(事实上, 模型确实可以仅从文本预训练中学到各地名的相对位置, 见[arXiv:2310.02207](https://arxiv.org/abs/2310.02207)).

这一模型也可以是聚类模型. 例如, 为了检验模型是否能学到不同食物的类别, 可以使用$C=\{苹果, 梨子, 牛肉, 宫保鸡丁, \dots\}$收集模型响应, 并观测在这些响应上运行的聚类是否与期望一致.

这一模型也可以是PCA. 对于一系列大致相似, 但只在某个概念/属性上不同的刺激(例如, 能吃或不能吃), 可以期望PCA所得到的主成分就反映了这个概念对应的表征空间中的方向(称作读出向量, reading vector). 在这些方向上操纵 token 的表示, 也就能操纵 token 的语义.

{% endspoiler %}

相应于层级 2 的实验, 从 LAT 导出的 reading vector 可以作为 LLM 行为的操控方法的基准. 进一步地, 可以提出具有更好性能的 LoRRA (Low-Rank Representation Adaption) 模型操纵方法. 可以预期, 基于可解释性的模型操纵方法相比"将有关的操纵数据放入数据集微调模型"的范式有潜在的优越性. 首先, 这样的方法具有更高的精确度上限, 能只影响 LLM 的小部分行为; 其次, 能更为有效而完整地处理模型的行为; 常规的微调有灾难性遗忘的问题, 而这样的方法能通过仅影响模型表征中的少数维度(理想情况下), 来缓解这一问题.

{% spoiler Low-Rank Representation Adaption %}

首先定义所谓 contrast vector. constrast vector 是两组相对比的提示导致的表征的差.

在取得 constrast vector 后, 为了操控模型的行为, 可以使用常规的 LoRA 微调方法. 具体来说, 从模型的最浅层开始, 计算目标表征, 即使用 constrast vector 或 reading vector 对原表征的某种操作(见下文). 损失函数则为每层目标表征与模型实际产生的表征之间的 $L^2$ 范数的和.

完成微调后, LoRA 方法将提供一组低秩的参数增量.

{% endspoiler %}

在得到 reading vector/constrast vector 后(统一记为 $v$), 可以施行多种操作. 这些操作可以在 LoRRA 方法中产生目标表征. 操作的强度调节可以通过根据超参数放大或缩小 $v$ 来实现.

1. 直接将增量添加在模型表征上: $R\rightarrow R\pm v$. 这类似于施加刺激或抑制.
2. 分段操作: $R\rightarrow R + \text{sign}(R\cdot v)v$. 即, 增加表征在$v$方向上的绝对强度.
3. 投影操作: $R\rightarrow R - \frac{R\cdot v}{v^2}v$. 即, 完全消除表征在$v$方向上的分量.

第 3 和第 4 层级的实验主要用于说明所寻找到的表征与 LLM 行为有着充分且必要的关系.

### Example: Honesty

正确(truthfulness)的定义是模型的响应与事实一致. 诚实(honesty)的定义是模型的响应与模型内在的信念一致(即使模型的响应并不正确). RepE 的作者进行了如下实验设置: 考虑提示模板

```plaintext
USER: Pretend you're an {honest/untruthful} person making statements about the world. ASSISTANT: <statement>
```

其中, `<statement>` 是诸如 `The Earth's atmosphere protects us from harm` 之类的正确陈述. `{honest/untruthful}` 则分别提示模型进行诚实或不诚实的输出.

作者在构造提示后, 运行模型, 得到 PCA 方法所用的数据集; 通过利用数据集所对应的标签, 可以校正 PCA 方法导出的最大主成分, 即 reading vector 的正方向. 为了验证方法的正确性, 可以在测试集上取得`<statement>`部分某个 token 上的模型的表征, 并利用表征在 PCA 方法所得最大主成分方向上投影的符号来判定模型是否被要求说谎. 这一判定的正确率可以接近 1 (通过利用模型中间层提取的表征).

![收集模型响应的示意图. 相同刺激不同指令的提示对应token之间的差被用作PCA方法的输入](representation_collection.svg)

![对模型诚实度的可视化. 考虑到方法的输入是逐 token 采样的, 可以实现 token 级别的诚实度检测](lie.png)

### Example: Fact Editing

LLM 领域的可解释性的鼻祖之作之一是[Locating and Editing Factual Associations in GPT](https://rome.baulab.info/). 该工作提出了 Rank-One Model Editing (ROME) 方法, 将 LLM 的 FFN 层看作一个键值存储库, 即它将来自前一层的输入看作一个键, 寻找出存储库中最接近的键, 并输出对应的值; 随后通过修改该最接近的键所代表的值对应的 FFN 层权重, 来实现对模型中存储的事实的修改. 例如, 它可以让模型在本该输出 `Eiffel Tower is in Paris, France` 的情况下输出 `Eiffel Tower is in Rome, Italy`. 这一工作通过批量损坏网络状态-逐个恢复网络状态来定位关键权重背后的基本方法论仍然被可解释性领域广泛采用. 然而, 该工作的结果远非完美, 也受到了诸多批评. 例如,

+ 该方法不是双向的(Q: 这和逆转诅咒有什么联系?), 即对 `Eiffel Tower is located in ____` 生效的修改不会导致 `Rome has a tower called the ____` 也同步被修改.
+ 该方法更多的是修改 token 到概念的关联, 而非概念本身. 对于一对同义词中一者的改动不会影响到另一者.
+ 如果在模型修改后没有提及被涉及的 token, 则修改不会生效. 必须直接提及该 token 来"激活"修改.
+ 该方法会导致意想不到的副作用, 例如编辑后, 提示中提及 Eiffel Tower 时, 模型会更多地谈论 Rome.
+ 该方法会导致其它无关的事实也被修改. 例如模型会认为卢浮宫也在罗马.

无论如何, 把埃菲尔塔搬到罗马已经成为了可解释性与模型编辑领域的 MNIST. 遵照 RepE 的一般思想, 作者首先要求模型生成一组与 `Eiffel Tower is in Paris` 这一事实有关的陈述作为对照组. 随后, 在生成的陈述中将 `Paris` 替换为 `Rome`, 得到实验组. 对于这两组数据, 作者在关键 token `Pairs/Rome` 的位置上收集模型响应, 得到 reading vector. 为了对模型进行实际的修改, 作者选择将 reading vector 与模型的原始表征进行线性组合. 相比 ROME 方法的缺陷, RepE 方法所得的结果有所改善.

![左: 模型正确地回答了修改后的事实结果, 这一修改能影响后续的模型推理. 右: 模型提及某概念的倾向也能被控制](rome.png)

### Example: Memorization

*在回忆事实和胡编乱造之间控制模型是一个不用努力想都知道有意义的应用.* -- 鲁迅

遵照前文, 读者想必已经能领会到 RepE 实验的一般流程了. 在本实验中, 作者采用真正的名人名言和让 GPT-4 合成的名言/真正的文献(书籍, 诗歌)的开头和合成的开头作为实验组和对照组; 所得的 reading vector 就表示了控制模型精确回忆 token 序列和模仿性生成 token 序列的表征的区别, 可以用来降低模型回忆 token 序列的倾向. 实验结果见下图. 考虑到 ROME 的前车之鉴, 作者还以一组众所周知的历史事件作为评估集. 结果表明, 回忆抑制操作仅降低历史事件数据集上 1% 的准确率(97.2%->96.2%). 这说明所得的 reading vector 确实准确调控了模型的死记硬背行为, 而没有伤及模型的知识能力.

![在使用随机方向而不是所得 reading vector 的方向来控制模型, 或在增强模型回忆倾向的方向控制模型, 均不显著改变模型输出结果与原始 token 序列之间的完全匹配率(Exact Match, EM)和嵌入向量相似度(Embedding Similarity, SIM); 在减弱模型回忆倾向的方向控制模型, 则能使前两个指标大幅下降](quote.png)

## Summary and Outlook

RepE 方法毫无疑问取得了有意义的进展. 然而, 值得注意的是, 其成功的基本条件在于, LLM 在预训练中自发涌现出的表征具有稀疏线性结构(Q: Why?[^1]), 使得简单的基线方法就能取得良好的效果.

// TODO

<!-- ```DOT
digraph {
    node [shape = record;];
    splines = false;
    rankdir=LR
    subgraph Honest {
        honest_prompt [label = "Honest Prompt|...Pretend you're an |<instruct>honest| <mid>person ... ASSISTANT: |<token1>The |<token2>Earth's |<token3>atmosphere |<token4>protects |<token5>us |<token6>from |<token7>harm";];
        honest_layer_n [label = "layer n|...Pretend you're an |<instruct>honest| <mid>person ... ASSISTANT: |<token1>The |<token2>Earth's |<token3>atmosphere |<token4>protects |<token5>us |<token6>from |<token7>harm";];
        honest_layer_m [label = "layer m|...Pretend you're an |<instruct>honest| <mid>person ... ASSISTANT: |<token1>The |<token2>Earth's |<token3>atmosphere |<token4>protects |<token5>us |<token6>from |<token7>harm";];
        
        honest_prompt:instruct -> honest_layer_n:instruct -> honest_layer_m:instruct;
        honest_prompt:mid -> honest_layer_n:mid -> honest_layer_m:mid;
        honest_prompt:token1 -> honest_layer_n:token1 -> honest_layer_m:token1;
        honest_prompt:token2 -> honest_layer_n:token2 -> honest_layer_m:token2;
        honest_prompt:token3 -> honest_layer_n:token3 -> honest_layer_m:token3;
        honest_prompt:token4 -> honest_layer_n:token4 -> honest_layer_m:token4;
        honest_prompt:token5 -> honest_layer_n:token5 -> honest_layer_m:token5;
        honest_prompt:token6 -> honest_layer_n:token6 -> honest_layer_m:token6;
        honest_prompt:token7 -> honest_layer_n:token7 -> honest_layer_m:token7;
        
        honest_prompt:instruct -> honest_layer_n:token1 [constraint = false;];
        honest_prompt:instruct -> honest_layer_n:token2 [constraint = false;];
        honest_prompt:instruct -> honest_layer_n:token3 [constraint = false;];
        honest_prompt:instruct -> honest_layer_n:token4 [constraint = false;];
        honest_prompt:instruct -> honest_layer_n:token5 [constraint = false;];
        honest_prompt:instruct -> honest_layer_n:token6 [constraint = false;];
        honest_prompt:instruct -> honest_layer_n:token7 [constraint = false;];
        
        honest_layer_n:instruct -> honest_layer_m:token1 [constraint = false;];
        honest_layer_n:instruct -> honest_layer_m:token2 [constraint = false;];
        honest_layer_n:instruct -> honest_layer_m:token3 [constraint = false;];
        honest_layer_n:instruct -> honest_layer_m:token4 [constraint = false;];
        honest_layer_n:instruct -> honest_layer_m:token5 [constraint = false;];
        honest_layer_n:instruct -> honest_layer_m:token6 [constraint = false;];
        honest_layer_n:instruct -> honest_layer_m:token7 [constraint = false;];
        
        honest_prediction [label = "Model Prediction:|<token1>The |<token2>Earth's |<token3>atomsphere |<token4>protects...";];
        honest_layer_m:token1 -> honest_prediction:token1 [style = invis;];
        honest_layer_m:token1 -> honest_prediction:token2 [constraint = false;];
        honest_layer_m:token2 -> honest_prediction:token3 [constraint = false;];
        honest_layer_m:token3 -> honest_prediction:token4 [constraint = false;];
        
        honest_layer_n_gather [label = "gathered layer n outputs";];
        honest_layer_m_gather [label = "gathered layer m outputs";];
        honest_layer_n:token1 -> honest_layer_n_gather [constraint = false;];
        honest_layer_n:token2 -> honest_layer_n_gather [constraint = false;];
        honest_layer_n:token3 -> honest_layer_n_gather [constraint = false;];
        honest_layer_n:token4 -> honest_layer_n_gather [constraint = false;];
        honest_layer_n:token5 -> honest_layer_n_gather [constraint = false;];
        honest_layer_n:token6 -> honest_layer_n_gather [constraint = false;];
        honest_layer_n:token7 -> honest_layer_n_gather [constraint = true;];
        
        honest_layer_m:token1 -> honest_layer_m_gather [constraint = false;];
        honest_layer_m:token2 -> honest_layer_m_gather [constraint = false;];
        honest_layer_m:token3 -> honest_layer_m_gather [constraint = false;];
        honest_layer_m:token4 -> honest_layer_m_gather [constraint = false;];
        honest_layer_m:token5 -> honest_layer_m_gather [constraint = false;];
        honest_layer_m:token6 -> honest_layer_m_gather [constraint = false;];
        honest_layer_m:token7 -> honest_layer_m_gather [constraint = true;];
    }
    
    subgraph Untruthful {
        untruthful_prompt [label = "Untruthful Prompt|...Pretend you're an |<instruct>untruthful| <mid>person ... ASSISTANT: |<token1>The |<token2>Earth's |<token3>atmosphere |<token4>protects |<token5>us |<token6>from |<token7>harm";];
        untruthful_layer_n [label = "layer n|...Pretend you're an |<instruct>untruthful| <mid>person ... ASSISTANT: |<token1>The |<token2>Earth's |<token3>atmosphere |<token4>protects |<token5>us |<token6>from |<token7>harm";];
        untruthful_layer_m [label = "layer m|...Pretend you're an |<instruct>untruthful| <mid>person ... ASSISTANT: |<token1>The |<token2>Earth's |<token3>atmosphere |<token4>protects |<token5>us |<token6>from |<token7>harm";];
        
        untruthful_prompt:instruct -> untruthful_layer_n:instruct -> untruthful_layer_m:instruct;
        untruthful_prompt:mid -> untruthful_layer_n:mid -> untruthful_layer_m:mid;
        untruthful_prompt:token1 -> untruthful_layer_n:token1 -> untruthful_layer_m:token1;
        untruthful_prompt:token2 -> untruthful_layer_n:token2 -> untruthful_layer_m:token2;
        untruthful_prompt:token3 -> untruthful_layer_n:token3 -> untruthful_layer_m:token3;
        untruthful_prompt:token4 -> untruthful_layer_n:token4 -> untruthful_layer_m:token4;
        untruthful_prompt:token5 -> untruthful_layer_n:token5 -> untruthful_layer_m:token5;
        untruthful_prompt:token6 -> untruthful_layer_n:token6 -> untruthful_layer_m:token6;
        untruthful_prompt:token7 -> untruthful_layer_n:token7 -> untruthful_layer_m:token7;
        
        untruthful_prompt:instruct -> untruthful_layer_n:token1 [constraint = false;];
        untruthful_prompt:instruct -> untruthful_layer_n:token2 [constraint = false;];
        untruthful_prompt:instruct -> untruthful_layer_n:token3 [constraint = false;];
        untruthful_prompt:instruct -> untruthful_layer_n:token4 [constraint = false;];
        untruthful_prompt:instruct -> untruthful_layer_n:token5 [constraint = false;];
        untruthful_prompt:instruct -> untruthful_layer_n:token6 [constraint = false;];
        untruthful_prompt:instruct -> untruthful_layer_n:token7 [constraint = false;];
        
        untruthful_layer_n:instruct -> untruthful_layer_m:token1 [constraint = false;];
        untruthful_layer_n:instruct -> untruthful_layer_m:token2 [constraint = false;];
        untruthful_layer_n:instruct -> untruthful_layer_m:token3 [constraint = false;];
        untruthful_layer_n:instruct -> untruthful_layer_m:token4 [constraint = false;];
        untruthful_layer_n:instruct -> untruthful_layer_m:token5 [constraint = false;];
        untruthful_layer_n:instruct -> untruthful_layer_m:token6 [constraint = false;];
        untruthful_layer_n:instruct -> untruthful_layer_m:token7 [constraint = false;];
        
        untruthful_prediction [label = "Model Prediction:|<token1>The |<token2>Earth's |<token3>atomsphere |<token4>harms...";];
        untruthful_layer_m:token1 -> untruthful_prediction:token1 [style = invis;];
        untruthful_layer_m:token1 -> untruthful_prediction:token2 [constraint = false;];
        untruthful_layer_m:token2 -> untruthful_prediction:token3 [constraint = false;];
        untruthful_layer_m:token3 -> untruthful_prediction:token4 [constraint = false;];
        
        untruthful_layer_n_gather [label = "gathered layer n outputs";];
        untruthful_layer_m_gather [label = "gathered layer m outputs";];
        untruthful_layer_n:token1 -> untruthful_layer_n_gather [constraint = false;];
        untruthful_layer_n:token2 -> untruthful_layer_n_gather [constraint = false;];
        untruthful_layer_n:token3 -> untruthful_layer_n_gather [constraint = false;];
        untruthful_layer_n:token4 -> untruthful_layer_n_gather [constraint = false;];
        untruthful_layer_n:token5 -> untruthful_layer_n_gather [constraint = false;];
        untruthful_layer_n:token6 -> untruthful_layer_n_gather [constraint = false;];
        untruthful_layer_n:token7 -> untruthful_layer_n_gather [constraint = true;];
        
        untruthful_layer_m:token1 -> untruthful_layer_m_gather [constraint = false;];
        untruthful_layer_m:token2 -> untruthful_layer_m_gather [constraint = false;];
        untruthful_layer_m:token3 -> untruthful_layer_m_gather [constraint = false;];
        untruthful_layer_m:token4 -> untruthful_layer_m_gather [constraint = false;];
        untruthful_layer_m:token5 -> untruthful_layer_m_gather [constraint = false;];
        untruthful_layer_m:token6 -> untruthful_layer_m_gather [constraint = false;];
        untruthful_layer_m:token7 -> untruthful_layer_m_gather [constraint = true;];
    }
    diff_layer_n_gather [style=bold; label="difference between representation with\n same stimulus and different instructions at layer n"];
    diff_layer_m_gather [style=bold; label="difference between representation with\n same stimulus and different instructions at layer m"];
    untruthful_layer_n_gather -> diff_layer_n_gather [constraint = false;];
    honest_layer_n_gather -> diff_layer_n_gather [constraint = false;];
    untruthful_layer_m_gather -> diff_layer_m_gather [constraint = false;];
    honest_layer_m_gather -> diff_layer_m_gather [constraint = false;];
}
``` -->

[^1]: 也许和[这里](https://zhuanlan.zhihu.com/p/635860634)有点关系?以及[这里](https://arxiv.org/abs/2311.03658)
