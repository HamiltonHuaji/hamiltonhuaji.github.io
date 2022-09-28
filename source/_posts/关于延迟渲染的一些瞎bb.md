---
title: 关于延迟渲染的一些瞎bb
date: 2022-09-29 00:33:47
tags:
- Computer Graphics
---

笔者受到[这里](https://advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pdf)的启发, 试图对不同种类的延迟渲染进行区分和总结, 故作本文. 本文是对延迟渲染的一些并不全面的认识.

<!-- more -->
## 前向(Forward)渲染

<del>前向渲染就是早期OpenGL设计者期望人们使用的那种渲染方式</del>. 即, 顶点缓冲区中的顶点经过图元装配、光栅化等流程后产生片元, 然后片元着色器输出的深度与深度缓冲中已有的深度值进行比较, 根据比较结果和预设的行为来决定是否覆盖帧缓冲中的颜色值; 最终还留在帧缓冲中的颜色值就是未被遮挡从而应显示在屏幕上的像素的值.
