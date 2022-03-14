---
title: 如何编写OptiFine光影包(工具篇)
date: 2022-03-05 16:22:38
mathjax: true
tags:
- Computer Graphics
- Shaderpack
---

工欲善其事, 必先利其器. 私以为比起光影包的核心算法本身, 辅助性的代码以及工具代码是远为重要的. 好的工具集能帮助你快速定位 bug, 这在抓帧工具难以使用的情况下[^1]是极其重要的. 本文主要介绍笔者在开发过程中产出的工具代码.

<!-- more -->
## 代码提示、预处理与验证

一门编程语言能否让人用得顺手不仅仅看这门语言本身设计是否优秀, 还要看与这门语言配套的编辑环境是否强大易用. 很不幸, glsl 并不是这样一门语言. 例如, VS Code 上似乎并不存在一个足够强大的 glsl linting 扩展, 来支撑我们的代码提示/语法检查/风格检查等需求. 配合上 glsl 本身的运行环境, 从书写代码到报错并修正错误的反馈回路将无比冗长.

然而, 作为一门极其类似 C 语言的语言, glsl 本身便向我们揭示了借用 C 与 C++ 生态的一种可能. glm 这个库便是极好的一个开始. 除了 glsl 的 swizzling 这一特性之外, 它能模拟 glsl 中几乎所有的数学函数的特性. 我们只需要在 glm 的基础上更进一步, 将 swizzling 这类特性和 sampler2D 这类 glsl 的内置类型也模拟出来, 便能将 glsl 代码伪装成 C++ 代码, 送给 C++ 的 linter 等工具检查, 或使用 clang-format 格式化代码. 当然, 最终还是需要利用 Khronos Group 的 glslangValidator 完成正式的代码检查.

模拟前述特性并不需要完成其实现, 而只需要有一个头文件向相关工具描述各对象的类型. 笔者完成的头文件可见[GLSL.hpp](https://github.com/HamiltonHuaji/GLSL.hpp/blob/master/glsl.hpp), 将其像包含正常头文件一样包含进 glsl 代码中, 并修改编辑器语言模式为 C++ 即可.

然而, OptiFine 有一个特性是 `#include` 指令, 即允许着色器文件包含来自其它文件的代码. 这一特性是组织起有序的工程所必须的. 然而 glslangValidator 并不能方便地检查带有该指令的代码(它要求源码中添加 `#extension GL_ARB_shading_language_include : require` 语句, 而我们并不希望 OptiFine 看到这一行). 一个并不复杂的方法是自己写一个预处理器, 完成对文件包含的处理. 笔者的预处理器见 [build.py](https://gist.github.com/HamiltonHuaji/9cfcf2c223cde1d5f00d0d85e71bb9cc). 考虑到 OptiFine 有一些配置是通过在着色器源码中插入特定格式的注释来实现的(指定 MRT 的 render targets 时需在文件中写 `/* DRAWBUFFERS: XYZ */` 这样的注释), 而这种格式的要求较严格(对于空格和缩进不鲁棒), 且在其书写出错时没有报错提示, 这一预处理器还能将形如 `#pragma drawbuffers(XYZ)` 的表达式转换成前述形式, 以避免偶然写出错误的注释格式.

## Gamma 校正

当前主流显示器对输入像素值产生的亮度响应是非线性的. 即, 亮度 `$L$` 与输入的像素值 `$x$` 近似有关系

$$L = L(x) = C x ^{\text{Gamma}}$$

其中, `$\text{Gamma}$` 是一个在 `$2.2$` 附近的值, 随着显示器的不同而不同.

为了使显示器输出的亮度与像素值呈线性关系, 我们可以将送入显示器的亮度值改为 `$x^{\frac{1}{\text{Gamma}}}$`, 使这一非线性变换与显示器响应的非线性相抵消. 这就是 Gamma 校正.

Gamma 校正看似只影响最终的显示效果, 但在我们的开发过程中, 这一校正绝不是可有可无的.

![未经 Gamma 校正的图像. 左侧是未经降噪的光线追踪结果, 右侧是经过降噪的光线追踪结果. 左侧在视觉上明显具有更高的平均亮度](gamma_uncorrected.png)

如上图所示, 未经降噪的光线追踪结果将产生更高的表观亮度. 这一现象可以根据 `$L(x)$` 是一个凸函数这一点而得到解释[^2]. 人眼感受到的亮度是图片上的亮度的时间平均值. 因此由琴生不等式(Jensen's inequality), 有

$$\overline{L(x)} \geq L(\overline{x})$$

其中, `$x$` 表示某点的未经降噪的光线追踪的结果的像素值, 是一个随机变量; 所谓降噪, 就是求取这个随机变量的期望值, 或用另一个方差更小的随机变量来代替 `$x$` 的过程. `$\overline{x}$` 表示 `$x$` 的平均值. 因此上述不等式左侧的含义是未经降噪的光线追踪的结果的像素值, 传递给显示器后得到的亮度的时间平均; 右侧的含义是光线追踪的结果的像素值的时间平均值传递给显示器后得到的亮度. 这一不等式说明在**未经 Gamma 校正的情况下, 未经降噪的光线追踪的结果看起来总是更亮一些**(在 `$x$` 有分布的情况下, 等号不会被取到), **就会在调试降噪算法的过程中得到极有误导性的结果**.

那么如何进行 Gamma 校正呢? 最直接的办法(然而也很粗糙)就是在 `$2.2$` 附近一个个 Gamma 值地去试. 好在 OptiFine 给我们提供了还不错的菜单功能, 能够大大加快试 Gamma 值的过程. 我们可以制作一个校正用的简单 shaderpack[^3].

将以下代码保存为 `shaders/final.fsh`
```glsl
#version 430 compatibility
#define GAMMA 1.9 // [0.5 1.0 1.5 1.6 1.7 1.8 1.9 2.0 2.1 2.2 2.3 2.4 2.5 3.0 3.5 4.0]
uniform float viewWidth;
uniform float viewHeight;
vec2 viewSize = vec2(viewWidth, viewHeight);
vec2 texCoord = gl_FragCoord.xy / viewSize;
ivec2 texelPos = ivec2(gl_FragCoord.xy);

void main() {
    vec4 finalColor;
    if (texCoord.x < .5) {
        if (((texelPos.x ^ texelPos.y) & 1) > 0) {
            finalColor = vec4(pow(vec3(0,0,0), vec3(1.f / GAMMA)), 1.f);
        } else {
            finalColor = vec4(pow(vec3(1,1,1), vec3(1.f / GAMMA)), 1.f);
        }
    } else {
        finalColor = vec4(pow(vec3(.5, .5, .5), vec3(1.f / GAMMA)), 1.f);
    }
    gl_FragData[0] = vec4(pow(finalColor.rgb, vec3(1.f / GAMMA)), 1);
}
```

将以下代码保存为 `shaders/shaders.properties`
```properties
screen =GAMMA
sliders=GAMMA
```

此时, 你应当有一个看起来长这样的目录(假设 `Gamma` 是你进行这些工作的目录):
```text
Gamma/
└── shaders
    ├── final.fsh
    └── shaders.properties
```

通过将整个 `Gamma` 文件夹(或打成 zip 包后)挪到你平常存放光影包的地方, 并在 OptiFine 里启用这个光影包, 你应当看到屏幕左侧黑像素和白像素构成非常小的棋盘格, 右侧是纯灰色. 摘下眼镜(如果你近视的话), 或坐到距屏幕较远的地方, 在屏幕左侧的棋盘格看不清楚时, 你就能开始 Gamma 校正.

在你显示器的 Gamma 值不是正好 1.9 的情况下, 你会感到屏幕左侧和右侧的亮度有区别. 进入 minecraft 的 `设置-光影-Gamma(即我们的校正 shaderpack 的名字)-光影设置` 界面, 你应当能看到有一个滑动条写着 `GAMMA`. 如果你感到屏幕左侧更亮, 就把滑动条往右挪; 反之往左挪, 直到屏幕两侧的亮度看起来近似相等为止. 这时滑动条上的数值就是你的显示器的 Gamma 值. 在光影包的 `final.fsh` 中, 需要根据我们测出的 Gamma 值, 按照本节开头给出的变换, 给输出到屏幕的颜色值完成 Gamma 校正.

// To be continued...

[^1]: Nvidia 家的 Nsight 会在抓帧后卡死游戏, 因此每启动一次游戏只能抓到一帧; 而由于 minecraft 使用的 OpenGL 版本过于奇怪, RenderDoc 也无能为力.
[^2]: 以下的解释忽略数学上的严谨性, 并跳过推导.
[^3]: 光影包和 shaderpack 是同一个东西, 因为这个校正光影包过于简陋, 对不起光影包这个名字, 就换了个叫法.
