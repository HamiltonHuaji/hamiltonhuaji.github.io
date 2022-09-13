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
即可生成供 IDE 跳转的未混淆的源码包. 这一源码包的文件位置在`.gradle/loom-cache/1.18.2/`下, 是一个包含了一大堆 `.class` 文件的 jar 包.

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

{% spoiler net/minecraft/client/render/WorldRender.java:263 %}
```java
    public WorldRenderer(MinecraftClient client, BufferBuilderStorage bufferBuilders) {
        this.client = client;
        this.entityRenderDispatcher = client.getEntityRenderDispatcher();
        this.blockEntityRenderDispatcher = client.getBlockEntityRenderDispatcher();
        this.bufferBuilders = bufferBuilders;
        for (int i = 0; i < 32; ++i) {
            for (int j = 0; j < 32; ++j) {
                float f = j - 16;
                float g = i - 16;
                float h = MathHelper.sqrt(f * f + g * g);
                this.field_20794[i << 5 | j] = -g / h;
                this.field_20795[i << 5 | j] = f / h;
            }
        }
        this.renderStars();
        this.renderLightSky();
        this.renderDarkSky();
    }
```
{% endspoiler %}
以上是 `WorldRenderer` 的构造函数. 考虑到构造函数只执行一次, 而熟悉 OpenGL 的同学可能会认为渲染相关的逻辑需要每帧都执行, 函数体末尾三个 `render*` 值得解释一下其作用. 以 `renderStars()` 函数为例:

{% spoiler net/minecraft/client/render/WorldRender.java:577 %}
```java
    private void renderStars() {
        Tessellator tessellator = Tessellator.getInstance();
        BufferBuilder bufferBuilder = tessellator.getBuffer();
        RenderSystem.setShader(GameRenderer::getPositionShader);
        if (this.starsBuffer != null) {
            this.starsBuffer.close();
        }
        this.starsBuffer = new VertexBuffer();
        this.renderStars(bufferBuilder);
        bufferBuilder.end();
        this.starsBuffer.upload(bufferBuilder);
    }
```
{% endspoiler %}
容易看出, 这一函数的实际作用是填充了 `this.starsBuffer` 背后的顶点缓冲的内容. 

通过搜索 `starsBuffer` 的用法, 可以发现 net/minecraft/client/render/WorldRender.java:1596 的 `public void renderSky(MatrixStack...)` 函数实际上执行了星空的绘制.

此处还出现了一个常见的类: BufferBuilder. 参考 `private void renderStars(BufferBuilder buffer)` 函数, 可以得到 BufferBuilder 的主要使用模式:

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

// TODO
