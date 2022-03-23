---
title: 如何编写OptiFine光影包(1)
date: 2022-03-14 16:32:47
mathjax: true
tags:
- Computer Graphics
- Shaderpack
---

这是《如何编写OptiFine光影包》系列正文的第一篇. 本篇教程将搭建光影包编写的基本框架, 以接管 OptiFine/minecraft 自带的那些着色器, 将延迟着色真正地执行起来.

<!-- more -->
作为教学性质的光影包, 笔者期望能在教程结束时实现以下特性和指标:

1. 光线追踪每帧样本量 1~4 spp
2. 使用 SVGF 降噪
3. 能处理透明物体的反射, 但出于简化的考虑, 假定所有折射率都是 $1$; 递归的反射也不会被考虑
4. 实体受光照影响, 但不遮挡除太阳和月亮外的光源的光照, 来自实体的反射光也不会照亮其它物体
5. 不使用泛光等光污染后处理特效
6. 仅处理 `LD*E` 和 `LD*SE` 的路径[^1]

尽管更高的光线追踪性能是可以期望的, 但降噪算法也具有不小的消耗(SVGF 所使用的滤波器大小是 $5\times 5$ 的, 且要在不同的尺度上执行 5 次), 因此指标(1)的 1~4 spp 是较为合理的结果. 特性(3)是为了 TAA 方便而设置的, 这使得透明物体的透射能较为简单地执行时间重投影, 否则就需要考虑 [Temporally Reliable Motion Vectors for Real-time Ray Tracing
](https://sites.cs.ucsb.edu/~lingqi/publications/paper_trmv.pdf) 的方法, 何况 SEUS PTGI 也是这么干的(逃. 特性(4)使得光线追踪的求交完全不用考虑非方块形状的实体, 大大简化了求交算法, 而太阳/月亮的直接光照可以较方便地利用阴影映射处理. 特性(6)是我们能处理的较简单情况, 然而这使得我们无法计算[焦散](https://en.wikipedia.org/wiki/Caustic_(optics))等效果而只能做一些看起来比较像焦散的特效, 但 SEUS PTGI 也是这么干的×2... 何况焦散这样的效果本来就需要光子映射这样的技术去计算.

![minecraft RTX 的折射. 即使是 Nvidia, 也难以将复杂的反射和折射处理得完美无瑕](artifacts.png)

回顾[原理篇](https://hamiltonhuaji.github.io/2022/03/05/%E5%A6%82%E4%BD%95%E7%BC%96%E5%86%99OptiFine%E5%85%89%E5%BD%B1%E5%8C%85-%E5%B7%A5%E5%85%B7%E7%AF%87/)中的光影包目录结构, 笔者创建了下列目录树:
{% spoiler HowToOptiFine/commit/a2208932f54e50cb434cf52aa9467312b2f7f054 %}
```text
.
├── makefile
├── src
│   ├── block.properties
│   ├── composite.fsh
│   ├── composite15.fsh
│   ├── composite2.fsh
│   ├── core
│   │   ├── gbuffers
│   │   │   ├── gbuffers_discard.frag
│   │   │   ├── gbuffers_discard.vert
│   │   │   ├── gbuffers_main.frag
│   │   │   └── gbuffers_main.vert
│   │   ├── shadow
│   │   └── svgf
│   │       ├── flush_history_buffers.hpp
│   │       └── temporal_accumulation.hpp
│   ├── inc
│   │   ├── blockmapping.hpp
│   │   ├── buffers.hpp
│   │   ├── constants.hpp
│   │   ├── gbuffer.hpp
│   │   ├── glsl.hpp
│   │   ├── random.hpp
│   │   ├── uniforms.hpp
│   │   └── utils.hpp
│   ├── lang
│   │   └── zh_CN.lang
│   └── shaders.properties
└── tools
    ├── blocks.txt
    ├── build.py
    ├── genblocks.py
    ├── patch.py
    └── validate.sh

8 directories, 26 files
```
{% endspoiler %}

通过 `tools` 目录里的各种工具, 可以生成最终的 `shaders/` 目录. 乍一看, 有各式各样的 `gbuffers_*.*sh` 着色器需要编写, 非常可畏, 然而 `gbuffers_*.*sh` 可以大致分为 3 类, 即绘制透明物体的着色器①, 绘制不透明的普通物体的着色器②和绘制特效或云/日月等不受光照影响的物体的着色器③, 每类着色器的代码都是可以共用的.

对于③, 在教程的本阶段, 简单地丢弃它们的像素即可:
{% spoiler HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_discard.vert %}
{% ghcode https://github.com/HamiltonHuaji/HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_discard.vert {cap:false,lang:cpp} %}
{% endspoiler %}
{% spoiler HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_discard.frag %}
{% ghcode https://github.com/HamiltonHuaji/HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_discard.frag {cap:false,lang:cpp} %}
{% endspoiler %}
透明物体(如水面, 玻璃)的处理较为复杂, 和实体一起暂时归到这一类中.

工具篇提供的预处理工具会保证带有 `#pragma once` 指令的文件只被包含一次, 可以起到[保护宏](https://en.wikipedia.org/wiki/Include_guard)的作用. `inc/glsl.hpp` 是工具篇中用于将 glsl 伪装成 C++ 的头文件.

对于②, 我们需要绘制一张 G-buffer, 并在 composite 阶段完成着色:
{% spoiler HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_main.vert %}
{% ghcode https://github.com/HamiltonHuaji/HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_main.vert {cap:false,lang:cpp} %}
{% endspoiler %}
{% spoiler HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_main.frag %}
{% ghcode https://github.com/HamiltonHuaji/HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/core/gbuffers/gbuffers_main.frag {cap:false,lang:cpp} %}
{% endspoiler %}

G-buffer 的定义如下
{% ghcode https://github.com/HamiltonHuaji/HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/inc/gbuffer.hpp 10 29 {cap:false,lang:cpp} %}

其打包的数据有:

+ `vec3 diffuse`, 为漫反射颜色
+ `vec3 normal`, 世界坐标下的法线
+ `vec3 tangent`, 世界坐标下的切线
+ `float smoothness`, 光滑度
+ `float metalness`, 金属度
+ `int blockID`, 16 bits 的方块 ID, 低 5 位表示方块形状, 第 6 位表示是否发光, 第 7 位表示是否透明, 高 8 位编码方块的各种数据值

最终 G-buffer 将被打包进 RGBA32F 格式(即有 RGBA 共 4 个通道, 每个通道都是 32 bits 的 float)的 `colortex0`.

世界坐标将被写入同样是 RGBA32F 的 `colortex1`, 浪费了一个 w 分量, 日后不够用了再来优化.

`colortex4` 则是运动向量(motion vector), 此处我们学习 UE4 的做法, 在 G-buffer 阶段分别使用本帧和上帧的 MVP 矩阵, 计算好裁切坐标系中的坐标并作差, 得到运动向量(其 xy 分量是屏幕 uv 的变化量, z 分量是深度值的变化量). 在教程的本阶段就准备好运动向量的原因是希望较早地完成 TAA 的调试. 当然, 此处又有待填的坑, 即日后加入实体的渲染时, 还需要考虑实体的运动.

我们计划在 `composite.fsh` 中完成蒙特卡洛光线追踪, 而目前只是将 G-buffer 的 diffuse 部分加上噪声填到 `colortex6` 中, 以使 TAA 的调试能进行.
{% spoiler HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/composite.fsh %}
{% ghcode https://github.com/HamiltonHuaji/HowToOptiFine/blob/a2208932f54e50cb434cf52aa9467312b2f7f054/src/composite.fsh {cap:false,lang:cpp} %}
{% endspoiler %}

// To be continued...

[^1]: 路径表示法见[光传输](https://hbbalfred.github.io/2015/11/20/light-transport.html), 即用 `L` 表示光源, `D` 表示漫反射面, `S` 表示光泽反射面, `E` 表示眼睛.
