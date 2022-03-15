---
title: 如何编写OptiFine光影包-(1)
date: 2022-03-14 16:32:47
mathjax: true
tags:
- Computer Graphics
- Shaderpack
---

这是《如何编写OptiFine光影包》系列正文的第一篇. 本篇教程将对延迟渲染中各 pass 的职能进行分配, 并搭建光影包编写的基本框架, 以接管 OptiFine/minecraft 自带的那些着色器.

<!-- more -->
作为教学性质的光影包, 笔者期望能在教程结束时实现以下特性和指标:
1. 光线追踪每帧样本量 1~4 spp
2. 使用 SVGF 降噪
3. 能处理透明物体的反射, 但出于简化的考虑, 假定所有折射率都是 $1$
4. 实体受光照影响, 但不遮挡除太阳和月亮外的光源的光照, 来自实体的反射光也不会照亮其它物体

尽管更高的光线追踪性能是可以期望的, 但降噪算法也具有不小的消耗(SVGF 所使用的滤波器大小是 `$5\times 5$` 的, 且要在不同的尺度上执行 5 次), 因此指标(1)的 1~4 spp 是较为合理的结果. 特性(3)是为了 TAA 方便而设置的, 这使得透明物体的透射能较为简单地执行时间重投影, 否则就需要考虑 [Temporally Reliable Motion Vectors for Real-time Ray Tracing
](https://sites.cs.ucsb.edu/~lingqi/publications/paper_trmv.pdf) 的方法, 何况 SEUS PTGI 也是这么干的(逃. 特性(4)使得光线追踪的求交完全不用考虑非方块形状的实体, 大大简化了求交算法, 而太阳/月亮的直接光照可以较方便地利用阴影映射处理.

回顾[原理篇](https://hamiltonhuaji.github.io/2022/03/05/%E5%A6%82%E4%BD%95%E7%BC%96%E5%86%99OptiFine%E5%85%89%E5%BD%B1%E5%8C%85-%E5%B7%A5%E5%85%B7%E7%AF%87/)中的光影包目录结构, 不难看出, 一个具有雄心壮志而接手所有物体的渲染的光影包作者应创建下列目录树:

<details>
<summary>示例目录树</summary>
```text
./
├── src
│   ├── block.properties
│   ├── shaders.properties
│   ├── shadow.vsh
│   ├── shadow.fsh
│   ├── shadow.gsh
│   ├── gbuffers_armor_glint.fsh
│   ├── gbuffers_armor_glint.vsh
│   ├── gbuffers_basic.fsh
│   ├── gbuffers_basic.vsh
│   ├── gbuffers_beaconbeam.fsh
│   ├── gbuffers_beaconbeam.vsh
│   ├── gbuffers_block.fsh
│   ├── gbuffers_block.vsh
│   ├── gbuffers_clouds.fsh
│   ├── gbuffers_clouds.vsh
│   ├── gbuffers_damagedblock.fsh
│   ├── gbuffers_damagedblock.vsh
│   ├── gbuffers_entities.fsh
│   ├── gbuffers_entities.vsh
│   ├── gbuffers_entities_glowing.fsh
│   ├── gbuffers_entities_glowing.vsh
│   ├── gbuffers_hand.fsh
│   ├── gbuffers_hand.vsh
│   ├── gbuffers_handheldwater.fsh
│   ├── gbuffers_handheldwater.vsh
│   ├── gbuffers_item.fsh
│   ├── gbuffers_item.vsh
│   ├── gbuffers_line.fsh
│   ├── gbuffers_line.vsh
│   ├── gbuffers_skybasic.fsh
│   ├── gbuffers_skybasic.vsh
│   ├── gbuffers_skytextured.fsh
│   ├── gbuffers_skytextured.vsh
│   ├── gbuffers_spidereyes.fsh
│   ├── gbuffers_spidereyes.vsh
│   ├── gbuffers_terrain.fsh
│   ├── gbuffers_terrain.vsh
│   ├── gbuffers_textured.fsh
│   ├── gbuffers_textured.vsh
│   ├── gbuffers_textured_lit.fsh
│   ├── gbuffers_textured_lit.vsh
│   ├── gbuffers_weather.fsh
│   ├── gbuffers_weather.vsh
│   ├── deferred.fsh
│   ├── deferred1.fsh
│   ├── deferred2.fsh
│   ├── ...
│   ├── deferred15.fsh
│   ├── gbuffers_water.fsh
│   ├── gbuffers_water.vsh
│   ├── composite.fsh
│   ├── composite1.fsh
│   ├── composite2.fsh
│   ├── composite3.fsh
│   ├── ...
│   ├── composite14.fsh
│   ├── composite15.fsh
│   └── final.fsh
```
</details>

通过工具篇提供的预处理工具, 可以生成最终的 `shaders/` 目录. 这样的目录乍一看非常可畏, 然而 `gbuffers_*.*` 可以大致分为 3 类, 即绘制透明物体的着色器, 绘制不透明的普通物体的着色器和绘制特效/云/日月而暂时可以不处理的着色器.

// To be continued...
