---
title: 如何编写OptiFine光影包(原理篇)
date: 2022-03-03 23:48:03
mathjax: true
tags:
- Computer Graphics
- Shaderpack
---

本文将首先简要回顾蒙特卡洛光线追踪算法, 随后概述 OptiFine 的工作方式, 对该算法在光影包中的实现难点进行分析, 最后对主流光线追踪光影包的实现原理进行解释.

<!-- more -->
## 蒙特卡洛光线追踪

*本节仅为回顾性的叙述; [前人之述](http://graphics.stanford.edu/courses/cs348b-01/course29.hanrahan.pdf)备矣.*

1986年, Kajiya等人提出了渲染方程, 描述几何光学近似下光与物体表面的交互过程(各符号的解释见[渲染方程](https://zh.wikipedia.org/zh-hans/渲染方程)).

`$L_{\text{out}}(\vec{x}, \vec{\omega}) = \int_{\text{Hemisphere}} f_r(\vec{x}, \vec{\omega'}, \vec{\omega}) L_{\text{in}}(\vec{x}, \vec{\omega'}) (\omega' \cdot \vec{n}) d\vec{\omega'}$`

这一方程实际上还有一个隐藏的补充条件, 即设 `$\vec{x'}$` 是从 `$\vec{x}$` 处沿 `$\vec{\omega}$` 方向发出的光线的路径上的一点, 则 `$\vec{x'}$` 处收到来自 `$\vec{\omega}$` 方向的辐照度(radiance) `$L_{\text{in}}(\vec{x'}, \vec{\omega})$` 等于 `$\vec{x}$` 处沿 `$\vec{\omega}$` 方向发出的光线的辐照度 `$L_{\text{out}}(\vec{x}, \vec{\omega})$`. 这个条件使 `$L$` 成为对光照强度最合适的量度, 因为无需考虑传播过程中 `$L$` 的变化: 沿着同一束光, `$L$` 是始终保持相等的.

作为一个积分方程, 该方程一般不具有封闭形式的解, 因此需要通过蒙特卡洛方法, 求得一个随机变量`$L(\vec{\text{eye}}, \vec{\text{vision}})$`, 使其期望为方程的解或近似解.

考虑该积分方程的一个近似, 即光线在场景中反射`$N+1$`次后就会消失, 而未反射这么多次的光线则不受影响. 显然, 当`$N\rightarrow\infty$`时, 这一近似回到真正的渲染方程. 因此方程的解可以被写为`$2N$`维的积分. 设场景全部表面为`$\Sigma$`, 积分区间就是`$\Sigma^N$`(将渲染方程转化为按面积的积分后即可看出). 这样的高维积分正是蒙特卡洛方法所擅长的.

而蒙特卡洛方法在求解如上积分时的每一个采样点, 都等价于`$N$`条首尾相连的路径. 除最前一条路径的头端在光源上、最后一条路径的尾端在摄像机上以外, 每条路径的两端都在场景中的能反射光的表面上. 因此沿着这些路径求取该采样点对结果的贡献的过程, 就像追踪一条假想的光线路径一样. 这就是蒙特卡洛光线追踪名称的含义.

以下的伪代码可以作为蒙特卡洛光线追踪的一个典型示例(暂时不考虑镜面反射/透射/次表面散射, 只考虑粗糙表面的漫反射). `raytrace(r)`函数的期望值即为沿着假想光线`r`的方向看去得到的颜色.
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

## OptiFine能干些啥

[OptiFine](https://www.optifine.net/) 是一款较主流的 minecraft 图形优(魔)化(改)闭源mod. 一般所说的光影包实际上是包含了一些着色器源码和附带的美术资源的压缩包, 通过 OptiFine 替换 minecraft 原有的着色器而运行. 值得一提的是, OptiFine 有一个正在开发中的开源替代 [Iris](https://github.com/IrisShaders/Iris.git). 希望 Iris 能结束光影包开发文档稀缺的历史. 由于 minecraft 使用的图形栈是 OpenGL, 光影包中的着色器全部是基于 OpenGL 编写的.

此处我们不讨论 OptiFine 是如何对 minecraft 进行逻辑注入的, 而着眼于它组织各着色器、并提供给着色器必要的输入的方式.

以下为一个典型的 OptiFine 光影包的目录结构. 为了简明起见, 这一目录结构略去下界/末地等维度使用的着色器. 在下文中, 所提到的文件名均略去`shaders/`这一目录前缀.
```text
./
├── shaders
│   ├── block.properties
│   ├── shaders.properties
│   ├── shadow.vsh
│   ├── shadow.fsh
│   ├── shadow.gsh
│   ├── gbuffers_basic.fsh
│   ├── gbuffers_basic.vsh
│   ├── ...
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

OptiFine 的渲染管线是一个多 pass 的过程.  `pass` 这一名词指完成一次渲染的流程, 即从顶点输入开始, 经过光栅化, 深度测试等过程, 得到一帧数据的过程. 多 pass 的管线中, 前一 pass 的输出图像(颜色缓冲/深度缓冲等), 往往会成为随后 pass 的输入纹理. 以下为 OptiFine 提供的主要的纹理列表; 它们往往有一些别名, 然而从统一性考虑, 不妨忘掉这些别名.

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

OptiFine[^1] 将首先执行 shadow pass. 顾名思义, 这一步是为了生成阴影映射(shadow map), 为阴影的绘制提供便利. 这一 pass 执行的着色器是 `shadow.*sh`. OptiFine 将以太阳/月亮为视点, 视线指向当前摄像机[^2], 生成 MVP 矩阵. 为了保证不在屏幕可见范围内的物体也能投下阴影, 世界坐标内以摄像机为中心较大范围内的顶点均会被送入这一 pass 进行绘制. OptiFine 会首先绘制所有的不透明物体, 得到 `shadowcolor1` 和相应的深度缓冲 `shadowtex1`; 随后再叠加绘制水面等透明物, 得到 `shadowcolor0` 和 `shadowtex0`. 由于 `shadowtex*` 来自单通道的深度缓冲, 可以仅访问其 r 分量. `shadowcolor0/1` 的区别在需要获得半透明物体的阴影时是有意义的.[^3]

OptiFine 将随后执行第一个 gbuffer pass . 这一步的目的是生成延迟渲染所用到的 `G-buffer`. G-buffer 并不是一块特殊格式的缓冲区, 而是保存场景信息的颜色缓冲区的统称. 通过自行定义 G-buffer 数据的存储方式, 可以将感兴趣的场景信息, 如当前处理的片元的法线方向/世界坐标等, 编码到颜色值上, 写入如 `colortex*` 的颜色缓冲内, 并在完成遮挡剔除后, 传递给随后的着色器使用; 此后的着色器就只需处理屏幕上可见的物体表面, 从而大量地节约昂贵的颜色计算. 这就是延迟渲染方法的主要思想. 除了用于绘制水面/玻璃等半透明面的 `gbuffers_water.*sh` 以外的所有 `gbuffers_*.*sh` 是这一 pass 执行的着色器.

第一个 gbuffer pass 后执行的是 deferred pass. 将 `deferred.fsh` 视为 `deferred0.fsh`, 则 OptiFine 将按照 `deferredX.fsh` 的数字顺序依次执行这些着色器. 因此 deferred pass 实际上是很多个 pass 的总称. 有名言曰: "计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决"; 在我们讨论的语境下, 不妨说: "计算机图形学领域的任何问题都可以通过增加一个 pass 来解决". 前述的延迟渲染中的颜色计算一般在这一步完成, 这是因为这一步实际上是绘制了由两个三角形构成的足以覆盖整个屏幕的长方形, `deferred*.fsh` 就是它的片元着色器, 从而使 deferred pass 中着色器执行的每一个片元, 都能一一映射到屏幕上的一个像素. 在这里可以省去`deferred*.vsh`, 因为 OptiFine 会将默认的顶点着色器与我们书写的片段着色器组合到一起执行; 而在 deferred pass 中, OptiFine 提供的是平凡的顶点输入, 因此自行书写顶点着色器毫无意义. 蒙特卡洛算法是出了名的昂贵, 甚至需要设计专用硬件(对, 说的就是 Nvidia RTX 系列显卡的 RT Core)来运行. 延迟渲染的特性正适合处理这类算法. 将蒙特卡洛算法在 deferred pass (或随后的 composite pass)中执行, 就能保证每一个被处理的像素都能显示在屏幕上, 不浪费一丝一毫算力.

在 deferred pass 后执行的是第二个 gbuffer pass (原文档称作 GBuffers translucent ). 这一步执行 `gbuffers_water.*sh`, 绘制半透明面.

第二个 gbuffer pass 后是 composite pass. 这一 pass 和 deferred pass 非常类似, 只有执行时机的区别. 可以用来混合 gbuffer pass 2 中半透明面和其后的不透明物的颜色, 以完成彩色玻璃的视觉效果.

最后执行的是 final pass. 这一 pass 和 composite pass 实际上没有什么区别, 但执行的着色器是 `final.fsh`. 我们书写的片段着色器应将最终要输出到屏幕上的颜色填入 `gl_FragColor` 变量中, 或填入 `colortex0`.

## 蒙特卡洛光线追踪的绝望之谷

可以认为蒙特卡洛光线追踪算法是一种全局光照算法. 所谓全局光照, 就意味着场景中物体会互相影响彼此所受的光照. 这就是说, 当计算场景中任一表面所受的光照时, 就必须具有对场景中包括物体和光源在内的所有对象的访问能力. 对于蒙特卡洛光线追踪算法来说, 这一点体现在光线与场景求交(伪代码中 `intersect(r)` 函数干的事情)时需要查询场景中所有物体是否与光线相交.

然而, 前述的 OptiFine 渲染管线仍具有光栅化管线所受到的所有限制. 在顶点着色器中, 不能访问来自其它图元的顶点数据; 在片段着色器中, 能够访问的只有当前图元插值后的顶点数据以及 OptiFine 给定的那几张纹理. 此外, OptiFine 会进行一定的视锥剔除, 即所有不落在当前视角内的物体都有可能不被绘制, 从而完全消除了被顶点着色器和片段着色器处理的可能. 这一切限制仿佛断绝了全局光照算法的任何可能.

**但, 真的如此吗?**

## 主流光线追踪光**影**包的原理

光影包有个`影`字. 所以我们的答案要到阴影映射的来源, 即 shadow pass 中寻找. 由于以摄像机为中心的较大范围内的所有方块, 不论从屏幕看是否被遮挡, 均会被送到 shadow pass 中处理, 因此很适合使用 shadow pass 来保存场景内主要的几何体信息[^4].

很显然, shadow pass 只能影响 `shadowcolor*` 和 `shadowtex*` 的值. 那么我们就需要寻找一个方法将经过 shadow pass 的数据编码进这两张纹理. 出于节约显卡带宽的考虑[^5], 我们希望这一方法
1. 编码同样的场景需要的比特数尽量少
2. 每次向纹理写入时, 冗余的比特尽量少
3. 向纹理写入必然是并行的, 并行的时候不要发生竞态条件(或丢弃掉一部分数据)

感谢伟大的 Notch, minecraft 是一款体素游戏. 这意味着我们只需要知道场景中每个格子里的方块是什么就行了. 也就是说, 我们编码的粒度应当以方块为单位, 并且一个方块的数据占用的像素应尽量少(自然是只占用一个像素最好, 可行性另说), 即可满足要求(1)和要求(2). 由于一个格子里不可能有两个方块, 因此只要将每个方块的数据分别存储到纹理上不同位置的像素的颜色值上, 就能满足要求(3). 将方块的数据打包到颜色值这件事比较简单, 通过配置 OptiFine, 使 `shadowcolor*` 的格式为 `RGBA32F`, 你就有 `$128$` 个 bits 可供挥霍了[^6].

那么问题来了, 如何让着色器吃进一个方块(或者几个三角形), 吐出单个像素呢?

Shader Model 4.0引进了**几何着色器**(geometry shader), 并在 OpenGL 3.1 中得到支持. 今日的主流硬件皆实现了此特性. 几何着色器的执行时机夹在顶点着色器和片元着色器之间, 它接受来自顶点着色器的全部输出(即能访问构成当前图元的全部顶点), 并向片元着色器输出处理后的参数. 几何着色器的诸多神奇能力, 一言以蔽之, 即创建/删除/修改图元的能力.

以下为一段几何着色器的示例.
```glsl
#version 430 core
layout(triangles) in;
layout(triangle_strip, max_vertices=6) out;
in vec3 vertexPos[];
flat out vec3 centralPos;
void main() {
    centralPos = (vertexPos[0] + vertexPos[1] + vertexPos[2]) / 3;

    // 或 gl_Position = gl_in[0].gl_Position;
    gl_Position = vertexPos[0];
    EmitVertex();
    gl_Position = vertexPos[1];
    EmitVertex();
    gl_Position = vertexPos[2];
    EmitVertex();
    EndPrimitive();
    
    gl_Position = vertexPos[0] * vec3(-1, -1, 1);
    EmitVertex();
    gl_Position = vertexPos[2] * vec3(-1, -1, 1);
    EmitVertex();
    gl_Position = vertexPos[1] * vec3(-1, -1, 1);
    EmitVertex();
    EndPrimitive();
}
```
`layout(triangles) in;` 表示这一几何着色器接受三角形作为输入. `layout(triangle_strip, max_vertices=6) out;` 表示这一几何着色器输出三角形, 最多产生 6 个顶点. 与片元着色器相比, 该几何着色器接受的输入变量改为数组, 以顶点之序号为索引访问这些数组即可分别获取顶点着色器之输出. 每当调用 `EmitVertex();` 时, OpenGL 就会将当前标记为 `out` 的变量值以及 `gl_Position` 这类 OpenGL 内部变量打包为一个新的顶点. 当调用 `EndPrimitive();` 时, OpenGL 就会将此前打包的顶点组合成一个新的图元, 然后像普通的图元一样, 光栅化后交给片元着色器执行. 以上几何着色器的功能是在与原三角形相对屏幕的中心对称处多绘制一个旋转了 `$\pi\ \text{rad}$` 的三角形.

当几何着色器带有 `layout(points, max_vertices=1) out;` 声明时, 它就输出单个点作为图元. 这正是我们想要的[^7]. 通过在几何着色器中指定 `gl_Position = foo(blockCentralPosition);`, 即可在根据映射函数 `foo` 与 `blockCentralPosition` 相关的位置输出像素值. 映射函数 `foo` 有很多选择, 只要是一一的映射都是可以接受的[^8], 例如我的选择是
```glsl
const int voxelR = 128;
const int voxelD = 2 * voxelR;
const int voxelA = voxelD * voxelD;
const int voxelV = voxelD * voxelD * voxelD;

ivec2 getTexelPosFromVoxelPos(ivec3 voxelPos) {
    uvec3 uvoxelPos = clamp(uvec3(voxelPos + ivec3(voxelR, 0, voxelR)), uvec3(0), uvec3(voxelD - 1));
    return ivec2(uvoxelPos.xz + uvec2(mod(uvoxelPos.y, 16) * voxelD, floor(float(uvoxelPos.y) / 16.f) * voxelD));
}
```

*注: SEUS PTGI 的几何着色器使用方式略有不同, 因为它还同时生成了正常的 shadow map. 它的几何着色器会生成两个三角形, 第一个三角形的三个顶点在裁切坐标系中的同一位置, 也就是一个零尺寸的三角形, 最后也会只画一个像素点; 第二个三角形就是正常的三角形, 但进行了缩放和偏移, 使正常的那张 shadow map 只占用整个纹理的一部分*

在完成了以上的变种 shader pass 后, 即可在 deferred pass 等地方利用同样的映射函数 `foo` 访问 `shadowcolor*` 和 `shadowtex*`, 查询指定的空间位置是否有方块, 以及有方块时方块的种类等数据, 这是第一节伪代码中 `intersect(r)` 函数得以实现的基础. 余下的部分就是平平无奇(不)的蒙特卡洛光线追踪. 目前主流光线追踪光影包使用的求交算法多为[A Fast Voxel Traversal Algorithm for Ray Tracing](http://www.cse.yorku.ca/~amana/research/grid.pdf).



[^1]: 本文中对各 pass 的命名与 OptiFine 原文档略有不同, 是根据执行的着色器的文件名命名的.
[^2]: 即史蒂夫的眼睛.
[^3]: shadow pass 后实际上有两组类似于后文中 deferred pass 的着色器可以执行, 分别称作 shadow composite (执行 `shadowcomp*.*sh` )和 prepare (执行 `prepare*.*sh` ), 用得较少.
[^4]: 什么? 你问我 shadow pass 里也没有处理到的方块怎么办? 反正这样的方块离得够远, 影响不大, 就不用管了...
[^5]: 读写 G-buffer 时的显存带宽占用是延迟渲染典型的性能瓶颈. 很难相信读取带有类似 G-buffer 性质的我们的变种 shadow map 时显存带宽占用反而不重要了.
[^6]: 像素的深度值也是可以利用的, 即通过指定 `gl_Position.z` 的值, 这同样是一个 32 位的浮点数.
[^7]: 一个方块实际上有 2×6=12 个三角形, 因此朴素的写法会往 shadow map 上同一个位置写 12 次相同的值. 这可以通过在几何着色器上不发射重复写入的图元来实现优化.
[^8]: 很显然, 尝试利用 GPU 缓存的特性进行访存的局部性优化是好的想法; 或者尝试在 shadow map 上建一颗[八叉树](https://github.com/BruceKnowsHow/Octray.git)也不错.

// To be revised.
// Please copy or redistribute this article later, so as not to make your reprint contain wrong content.
