---
title: Minecraft渲染原理(1)
date: 2022-09-13 23:10:07
tags:
- Computer Graphics
- Minecraft
---

上回说到, 渲染一帧(包括绘制世界和绘制UI)的主要逻辑发生在 `net/minecraft/client/render/GamerRenderer.java` 中. 而实际上, `GamerRenderer.java` 又将渲染世界的任务分包给了 `net/minecraft/client/render/WorldRenderer.java`. 这两个类都不是一般的复杂, 因此我们可以尝试从一部分简单的逻辑入手, 来挖掘 ojng 程序员惯用的编码模式.

<!-- more -->
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

BufferBuilder 类本质上是对 ByteBuffer 的一个包装.
+ `bufferBuilder.begin()`可以指定绘制模式和顶点格式, 并将 bufferBuilder 置于准备接受新一个几何体的顶点数据的状态.
+ `bufferBuilder.vertex(x, y, z)` 会将 x,y,z 的值按顺序写入内置的 ByteBuffer, 并将内部状态设置为对下一个元素写入
+ `bufferBuilder.next()` 则会告知 bufferBuilder 完成对当前顶点的写入, 适当调整内部 ByteBuffer 的容量, 并准备好写入下一个顶点.
+ `bufferBuilder.end()` 则结束对当前顶点数据的收集, 根据一定条件补充索引数据(即供 OpenGL 使用的 index buffer), 并将收集到的绘制参数(例如顶点数、绘制模式、顶点格式等)打包, 压入 `bufferBuilder.parameters` 中, 随后重置 bufferBuilder 的状态.
BufferBuilder 是多个顶点数据和相关绘制参数构成的栈. `vertexBuffer.upload(bufferBuilder)` 则会弹出栈顶的数据, 并上传到其对应的 OpenGL 顶点缓冲区中.

{% spoiler 绘制模式(DrawMode)和顶点格式(VertexFormat)的详细说明 %}
绘制模式是顶点构成图元(Primitive)的方式. 例如, `VertexFormat.DrawMode.QUADS` 就是一种用于绘制四边形的绘制模式; 顶点流中相邻的 4 个顶点被分为一组(即一个图元), 构成一个 QUAD, 其中 (0,1,2), (2,3,0) 号顶点用于绘制两个三角形, 这两个邻接的三角形构成了期望得到的四边形. 这样的绘制模式的好处在于能节约顶点缓冲区中重复的顶点, 相比 `VertexFormat.DrawMode.TRIANGLE` (即相邻 3 个顶点构成一个三角形的绘制模式) 节约了 1/3 的空间. [除此之外](https://www.khronos.org/opengl/wiki/Primitive), 还有 TRIANGLE_FANS/TRIANGLE_STRIP 等绘制模式可用.

顶点格式就是对每个顶点包含的数据的布局的描述, 比如说如下的代码中
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/VertexFormats.java 13 36 {cap:false,lang:java} %}
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
{% endspoiler %}

// TODO
