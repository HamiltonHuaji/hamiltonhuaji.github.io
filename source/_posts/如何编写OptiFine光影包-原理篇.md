---
title: 如何编写OptiFine光影包(原理篇)
date: 2022-03-03 23:48:03
mathjax: true
tags:
- Computer Graphics
- Shaderpack
---

本文将首先简要回顾蒙特卡洛光线追踪算法, 随后结合 OptiFine 的工作方式, 对该算法在光影包中的实现难点进行分析, 最后对主流光线追踪光影包的实现原理进行解释.

## 蒙特卡洛光线追踪

1986年, Kajiya等人提出了渲染方程, 描述几何光学近似下光与物体表面的交互过程(各符号的解释见[渲染方程](https://zh.wikipedia.org/zh-hans/渲染方程))

`$L_{\text{out}}(\vec{x}, \vec{\omega}) = \int_{\text{Hemisphere}} f_r(\vec{x}, \vec{\omega'}, \vec{\omega}) L_{\text{in}}(\vec{x}, \vec{\omega'}) (\omega' \cdot \vec{n}) d\vec{\omega'}$`

作为一个积分方程, 该方程一般不具有封闭形式的解, 因此需要通过蒙特卡洛方法, 求得一个随机变量`$L(\vec{\text{eye}}, \vec{\text{vision}})$`, 使其期望为方程的解或近似解.

考虑该积分方程的一个近似, 即光线在场景中反射`$N+1$`次后就会消失, 而未反射这么多次的光线则不受影响. 显然, 当`$N\rightarrow\infty$`时, 这一近似回到真正的渲染方程. 因此方程的解可以被写为`$2N$`维的积分. 设场景全部表面为`$\Sigma$`, 积分区间就是`$\Sigma^N$`(将渲染方程转化为按面积的积分后即可看出). 这样的高维积分正是蒙特卡洛方法所擅长的.

而蒙特卡洛方法在求解如上积分时的每一个采样点, 都等价于`$N$`条首尾相连的路径. 除最前一条路径的头端在光源上、最后一条路径的尾端在摄像机上以外, 每条路径的两端都在场景中的能反射光的表面上. 因此沿着这些路径求取该采样点对结果的贡献的过程, 就像追踪一条假想的光线路径一样. 这就是蒙特卡洛光线追踪名称的含义.

以下的伪代码可以作为蒙特卡洛光线追踪的一个典型示例(暂时不考虑镜面反射/透射/次表面散射, 只考虑粗糙表面的漫反射)
```cpp
vec3 raytrace(Ray r) {
    IntersectData itd = intersect(r); // 沿着假想光线的方向与场景求交, 返回与假想光线相交的第一个表面的信息
    if (itd.isHit) {
        r.origin = itd.hitPosition;
        r.direction = generateRandomReflectDirection(itd.hitNormal, r.direction);
        if (itd.isEmissive) {
            return itd.emissiveColor + itd.reflectiveColor * raytrace(r);
        } else {
            return itd.reflectiveColor * raytrace(r);
        }
    } else {
        return vec3(0); // 没有触及任何表面
    }
}
```
然而, 这一代码很显然不适于用在任何实际的光线追踪渲染器上, 也有相当多的细节需要补充.

## OptiFine的工作方式

[OptiFine](https://www.optifine.net/) 是一款较主流的 minecraft 图形优(魔)化(改)闭源mod. 一般所说的光影包实际上是包含了一些着色器源码和附带的美术资源的压缩包, 通过 OptiFine 替换 minecraft 原有的着色器而运行. 值得一提的是, OptiFine 有一个正在开发中的开源替代 [Iris](https://github.com/IrisShaders/Iris.git). 希望 Iris 能结束光影包开发文档稀缺的历史.

此处我们不讨论 OptiFine 是如何对 minecraft 进行逻辑注入的, 而着眼于它组织各着色器、并提供给着色器必要的输入的方式.

以下为一个典型的 OptiFine 光影包的目录结构. 为了简明起见, 这一目录结构略去下界/末地等维度使用的着色器. 在下文中, 所提到的文件名均略去`shaders/`这一目录前缀.
```text
// TODO
```

OptiFine 的渲染管线是一个多 pass 的过程.  `pass` 这一名词指完成一次渲染的流程, 即从顶点输入开始, 经过光栅化, 深度测试等过程, 得到一帧数据的过程. 多 pass 的管线中, 前一 pass 的输出图像(颜色缓冲/深度缓冲等), 往往会成为随后 pass 的输入纹理. 以下为 OptiFine 提供的主要的纹理列表; 它们往往有一些别名, 但这些别名可能受到诸多因素的影响而指向不唯一, 因此不妨忘掉这些别名.

```glsl
uniform sampler2D shadowcolor0;
uniform sampler2D shadowtex0;

uniform sampler2D shadowcolor1;
uniform sampler2D shadowtex1;

uniform sampler2D noisetex;

uniform sampler2D depthtex0;
uniform sampler2D depthtex1;
uniform sampler2D depthtex2;

uniform sampler2D colortex0;
uniform sampler2D colortex1;
uniform sampler2D colortex2;
uniform sampler2D colortex3;
uniform sampler2D colortex4;
uniform sampler2D colortex5;
uniform sampler2D colortex6;
uniform sampler2D colortex7;
uniform sampler2D colortex8;
uniform sampler2D colortex9;
uniform sampler2D colortex10;
uniform sampler2D colortex11;
uniform sampler2D colortex12;
uniform sampler2D colortex13;
uniform sampler2D colortex14;
uniform sampler2D colortex15;
```

OptiFine 可以经过配置而将一部分着色器的输出指向特定纹理.

OptiFine 将首先执行 shadow pass. 顾名思义, 这一步是为了生成阴影映射(shadow map), 为阴影的绘制提供便利. 这一 pass 执行的着色器是 `shadow.*sh`. OptiFine 将以太阳/月亮为视点, 视线指向当前摄像机, 生成 MVP 矩阵. 为了保证不在屏幕可见范围内的物体也能投下阴影, 世界坐标内以摄像机为中心较大范围内的顶点均会被送入这一 pass 进行绘制. OptiFine 会首先绘制所有的不透明物体, 得到 `shadowcolor1` 和相应的深度缓冲 `shadowtex1`; 随后再叠加绘制水面等透明物, 得到 `shadowcolor0` 和 `shadowtex0`. 由于 `shadowtex*` 来自单通道的深度缓冲, 可以仅访问其 r 分量. `shadowcolor0/1` 的区别在需要获得半透明物体的阴影时是有意义的.

OptiFine 将随后执行 gbuffer pass. 这一步的目的是生成延迟渲染所用到的 `G-buffer`. G-buffer 并不是一块特殊格式的缓冲区, 而是保存场景信息的颜色缓冲区的统称. 通过自行定义 G-buffer 数据的存储方式, 可以将感兴趣的场景信息, 如当前处理的片元的法线方向/世界坐标等, 编码到颜色值上, 写入如 `colortex*` 的颜色缓冲内, 并在完成遮挡剔除后, 传递给随后的着色器使用; 此后的着色器就只需处理屏幕上可见的物体表面, 从而大量地节约昂贵的颜色计算. 这就是延迟渲染方法的主要思想. `gbuffers_*.*sh` 是这一 pass 执行的着色器.

gbuffer pass 后执行的是 composite pass. 前述的延迟渲染中的颜色计算一般在这一步完成. 将 `composite.fsh` 视为 `composite0.fsh`, 则 OptiFine 将按照 `compositeX.fsh` 的数字顺序依次执行这些着色器. 因此 composite pass 实际上是很多个 pass 的总称. 有名言曰: "计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决"; 在我们讨论的语境下, 不妨说: "计算机图形学领域的任何问题都可以通过增加一个 pass 来解决".

// To be continued...
