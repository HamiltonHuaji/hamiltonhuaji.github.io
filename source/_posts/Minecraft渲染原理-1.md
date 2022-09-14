---
title: Minecraft渲染原理(1)
date: 2022-09-13 23:10:07
tags:
- Computer Graphics
- Minecraft
---

上回说到, 渲染一帧(包括绘制世界和绘制UI)的主要逻辑发生在 `net/minecraft/client/render/GameRenderer.java` 中. 而实际上, `GameRenderer.java` 又将渲染世界的任务分包给了同一目录下的 `WorldRenderer.java`.

`GameRenderer.render` 函数完成的主要工作罗列如下:
<!-- more -->

+ 处理暂停相关逻辑, 必要时将 `MinecraftClient.currentScreen` 设置成暂停界面
+ 根据屏幕尺寸设置视口 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 824 824 {cap:false,lang:java} %}
+ 渲染世界 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 827 827 {cap:false,lang:java} %}
+ 将实体的轮廓线绘制到默认帧缓冲[^1] {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 829 829 {cap:false,lang:java} %}
+ 进行后处理[^2] {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 835 835 {cap:false,lang:java} %}
+ 绑定 `MinecraftClient.framebuffer` 对应的 FBO {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 837 837 {cap:false,lang:java} %}
+ 设置一番 `RenderSystem` 里的 MV 和 P 矩阵 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 841 846 {cap:false,lang:java} %}
+ 设置渲染 GUI 所需的一些 uniform (?), 似乎是为了产生光照效果 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 847 847 {cap:false,lang:java} %}
+ 渲染[反胃](https://minecraft.fandom.com/zh/wiki/%E5%8F%8D%E8%83%83)的特效(如果玩家处于相应状态的话) {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 853 853 {cap:false,lang:java} %}
+ 绘制悬浮项[^3]和 HUD {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 856 857 {cap:false,lang:java} %}
+ 绘制叠加层[^4] {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 864 864 {cap:false,lang:java} %}
+ 绘制 `MinecraftClient.currentScreen` {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 875 875 {cap:false,lang:java} %}

点进 `currentScreen` 所属的类 `Screen` 可以发现, 其 `render` 方法有十万个甚至九万个重写, 例如 `BeaconScreen`/`BookScreen`/`DeathScreen` 等类都继承了 `Screen` 类并重写了该方法, 令人望而生畏. 根据这些子类的名字可以判断出, 绘制 `currentScreen` 就是绘制游戏中的各种菜单和界面, 如信标选择效果的界面/打开书的界面/死亡重生界面等.

发光实体/后处理/反胃/不死图腾/叠加层等特效都不是我们所关心的, 只有 `GameRenderer.renderWorld` 函数里的东西才是热衷于烧显卡的程序员的热爱之物.

`GameRenderer.renderWorld` 干的事情如下:

+ 让 `GameRenderer.lightmapTextureManager` 进行某种更新. 鉴于根本没有正经光影包会管游戏里自带的光照图, 我们可以当这件事情不存在.
+ 设置相机依附的实体, 以合理更新后处理着色器(比如说蜘蛛) {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 978 980 {cap:false,lang:java} %}
+ 更新十字叉丝瞄准的实体 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 981 981 {cap:false,lang:java} %}
+ 进行了一堆神必的矩阵操作以获得 P 矩阵 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 987 1005 {cap:false,lang:java} %}
+ 更新摄像机的状态, 如依附的实体/旋转角/位置等量, 并进行了一堆神必的矩阵操作将摄像机的位姿加入 MV 矩阵 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 1006 1012 {cap:false,lang:java} %}
+ 设置 `WorldRenderer` 的视锥体, 并进行渲染(要来力! (狂喜)) {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 1013 1014 {cap:false,lang:java} %}
+ 绘制手(和手里拿着的物品) {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/GameRenderer.java 1016 1019 {cap:false,lang:java} %}

欲知 `WorldRenderer.render(MatrixStack ...)` 如何, 且听下回分解.

[^1]: 带有[发光](https://minecraft.fandom.com/zh/wiki/%E5%8F%91%E5%85%89)效果的实体,其特效就是通过这一步实现的
[^2]: 例如观察者模式下的蜘蛛/末影人/苦力怕视角
[^3]: floatItem似乎只出现在不死图腾被触发的时刻,因此可以猜测其内容就是不死图腾的特效
[^4]: 似乎只有一种overlay,即SplashOverlay,根据其引用的资源`textures/gui/title/mojangstudios.png`可以猜测是渲染启动游戏或重新载入时的红底白字界面
