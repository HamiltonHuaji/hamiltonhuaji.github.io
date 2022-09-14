---
title: Minecraft渲染原理(0)
date: 2022-09-13 14:57:59
tags:
---

首先简单介绍一下 Minecraft 源码的获取方式. fabric 开发环境包含了反混淆 Minecraft 源码的工具, 因此最简单的方法是克隆一个 fabric mod 的仓库, 然后调用里面的工具. 考虑到我们的目的是修改 VulkanMod, 不妨直接 `git clone https://github.com/xCollateral/VulkanMod.git` (或克隆 [fabric-example-mod](`https://github.com/FabricMC/fabric-example-mod`) 也可).

在获得了 VulkanMod 的代码仓库后, 在仓库文件夹内执行
```shell
./gradlew genSources
```
即可生成供 IDE 跳转的未混淆的源码包. 这一源码包的文件位置在`VulkanMod/.gradle/loom-cache/1.18.2/`下, 是一个包含了一大堆 `.class` 文件的 jar 包.

<!-- more -->
展开这个 jar 包(解压缩或者使用 jd-gui), 应当得到如下的目录树:

{% spoiler 目录树 %}
```text
.
├── META-INF
│   └── MANIFEST.MF
├── com
│   └── mojang
│       └── blaze3d
├── minecraft-project-merged-named-sources.iml
└── net
    └── minecraft
        ├── Bootstrap.java
        ├── GameVersion.java
        ├── MinecraftVersion.java
        ├── SaveVersion.java
        ├── SharedConstants.java
        ├── advancement
        ├── block
        ├── class_6148.java
        ├── class_6567.java
        ├── class_6955.java
        ├── client
        ├── command
        ├── data
        ├── datafixer
        ├── enchantment
        ├── entity
        ├── fluid
        ├── inventory
        ├── item
        ├── loot
        ├── nbt
        ├── network
        ├── obfuscate
        ├── particle
        ├── potion
        ├── predicate
        ├── recipe
        ├── resource
        ├── scoreboard
        ├── screen
        ├── server
        ├── sound
        ├── stat
        ├── state
        ├── structure
        ├── tag
        ├── test
        ├── text
        ├── unused
        ├── util
        ├── village
        └── world
```
{% endspoiler %}

而我们感兴趣的内容主要集中在 `net/minecraft/client/` 目录下. 通过搜索所有 java 人都熟悉的 `public static void main`, 我们知道可以从这一目录下的 `net/minecraft/client/main/Main.java` 开始梳理游戏客户端的运行逻辑.

{% spoiler net/minecraft/client/main/Main.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/main/Main.java 192 218 {cap:false,lang:java} %}
{% endspoiler %}

<del>这段代码充分展现了 ojng 招聘的员工素质, 通过死循环来等待另一个线程结束.</del> 事实上, `minecraftClient.shouldRenderAsync()` 永远是 `false`, 因此只有第二个分支会执行; 这导致主线程就是渲染线程.

我们可以据此跳转到
{% spoiler net/minecraft/client/MinecraftClient.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 706 740 {cap:false,lang:java} %}
{% endspoiler %}
724行的 `this.render(!bl)` 就是我们要找的目标. 这一函数实在太长, 我们只简单罗列正常渲染一帧时其所做事项, 至于退出/重载资源等行为先不管:

+ 执行 tick
+ 更新鼠标位置(用于gui展开的情况) {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1039 1039 {cap:false,lang:java} %}
+ 对 `RenderSystem.modelViewMatrix` 进行一番操作 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1045 1047 {cap:false,lang:java} %}
+ 绑定 `MinecraftClient.framebuffer` 对应的 FBO {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1049 1049 {cap:false,lang:java} %}
+ 清除雾 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1050 1050 {cap:false,lang:java} %}
+ 启用材质和剔除 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1052 1053 {cap:false,lang:java} %}
+ 调用 `gameRenderer` 进行真正的绘制 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1057 1057 {cap:false,lang:java} %}
+ 绘制 `toast` [^1] {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1059 1059 {cap:false,lang:java} %}
+ 绑定默认帧缓冲 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1068 1068 {cap:false,lang:java} %}
+ 将 `MinecraftClient.framebuffer` 对应的 FBO 的内容使用 `gameRenderer.blitScreenShader` 绘制到默认帧缓冲上 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1072 1072 {cap:false,lang:java} %}
+ 交换颜色缓冲, 使默认帧缓冲上的内容显示到屏幕上 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1076 1076 {cap:false,lang:java} %}
+ 等待合适的时间以使帧率不超过设定的限制 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/MinecraftClient.java 1079 1079 {cap:false,lang:java} %}

由上述过程可知, 一切的魔法都发生在 `gameRenderer.render` 中. 且听下回分解.

[^1]: toast是Minecraft在获得成就/获得新配方/完成教程时弹出的框框,大致外观可见`assets/minecraft/textures/gui/toasts.png`
