---
title: Minecraft渲染原理(1)
date: 2022-09-13 14:57:59
tags:
- Computer Graphics
- Minecraft
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

而我们感兴趣的内容主要集中在 `net/minecraft/client/render/` 目录下.

这一目录下, 有两个文件吸引了我们的注意力: `GamerRenderer.java` 和 `WorldRenderer.java`. 顾名思义, 这两个类包含了渲染游戏和渲染世界的主要逻辑.

{% spoiler net/minecraft/client/render/WorldRenderer.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 263 280 {cap:false,lang:java} %}
{% endspoiler %}
以上是 `WorldRenderer` 的构造函数. 考虑到构造函数只执行一次, 而熟悉 OpenGL 的同学可能会认为渲染相关的逻辑需要每帧都执行, 函数体末尾三个 `render*()` 值得解释一下其作用. 以 `renderStars()` 函数为例:

{% spoiler net/minecraft/client/render/WorldRenderer.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 577 588 {cap:false,lang:java} %}
{% endspoiler %}
容易看出, 这一函数的实际作用是填充了 `this.starsBuffer` 背后的顶点缓冲的内容. 

通过搜索 `starsBuffer` 的用法, 可以发现 net/minecraft/client/render/WorldRender.java:1596 的 `public void renderSky(MatrixStack...)` 函数实际上执行了星空的绘制.

此处还出现了一个在源码中很常见的类: BufferBuilder. 参考 `private void renderStars(BufferBuilder buffer)` 函数, 可以得到 BufferBuilder 的主要使用模式:

{% spoiler BufferBuilder usage pattern %}
```java
BufferBuilder bufferBuilder = new BufferBuilder();
VertexBuffer vertexBuffer = new VertexBuffer();

bufferBuilder.begin(VertexFormat.DrawMode.QUADS, VertexFormats.POSITION);
buffer.vertex(x, y, z).next();
...
buffer.end();
vertexBuffer.upload(buffer);
```
{% endspoiler %}

`bufferBuilder.begin()`可以指定绘制模式和顶点格式, 并将 bufferBuilder 置于准备接受新一个几何体的顶点数据的状态. 例如, `VertexFormat.DrawMode.QUADS` 就是一种用于绘制四边形的绘制模式; 缓冲区中 4 个顶点为一组构成一个 QUAD, 其中 (0,1,2), (0,2,3) 号顶点用于绘制两个三角形, 这两个有一条边相邻的三角形构成了期望的四边形. 这样的绘制模式的好处在于能节约顶点缓冲区中重复的顶点. 除此之外, 还有 TRIANGLE_FANS/TRIANGLE_STRIP 等绘制模式可用. 顶点格式就是对每个顶点包含的数据的布局的描述, 比如说如下的代码中
{% spoiler net/minecraft/client/render/VertexFormats.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/VertexFormats.java 13 36 {cap:false,lang:java} %}
{% endspoiler %}
`VertexFormats.POSITION_ELEMENT` 表示由 3 个 FLOAT 构成的元素, 用于传递坐标值; `VertexFormats.COLOR_ELEMENT` 表示由 4 个 UBYTE 构成的元素, 用于传递顶点色.

`VertexFormats.POSITION` 表示一种只包含坐标值的顶点格式, 坐标值绑定到 `Position` 上, 对应于以下着色器声明:
```glsl
layout(location = 0) in vec3 Position;
```
`VertexFormats.POSITION_COLOR` 表示既包含坐标元素又包含顶点色元素的顶点格式, 分别绑定到 `Position` 和 `Color` 上, 对应于以下着色器声明:
```glsl
layout(location = 0) in vec3 Position;
layout(location = 1) in vec4 Color;
```

BufferBuilder 类本质上是对 ByteBuffer 的一个包装. `bufferBuilder.vertex(x, y, z)` 会将 x,y,z 的值按顺序写入内置的 ByteBuffer, 并将内部状态设置为对下一个元素写入; `bufferBuilder.next()` 则会告知 bufferBuilder 完成对当前顶点的写入, 适当调整内部 ByteBuffer 的容量, 并准备好写入下一个顶点. `bufferBuilder.end()` 则结束对当前顶点数据的收集, 根据一定条件补充索引数据(即供 OpenGL 使用的 index buffer), 并将收集到的绘制参数(例如顶点数、绘制模式、顶点格式等)打包, 压入 `bufferBuilder.parameters` 中, 随后重置 bufferBuilder 的状态.

根据上述说明, 可以知道 BufferBuilder 是多个顶点数据和相关绘制参数构成的栈. `vertexBuffer.upload(bufferBuilder)` 则会弹出栈顶的数据, 并上传到其对应的 OpenGL 顶点缓冲区中.
