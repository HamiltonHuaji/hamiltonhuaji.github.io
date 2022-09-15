---
title: Minecraft渲染原理(2)
date: 2022-09-14 17:15:30
tags:
- Computer Graphics
- Minecraft
---

上回说到, 对世界的渲染被分包到了 `WorldRenderer.render(MatrixStack ...)` 这个函数. 这是一个将近 300 行的巨大函数, <del>再次展现了 ojng 招聘的员工素质.</del>

{% spoiler net/minecraft/client/render/WorldRenderer.java:992 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 992 1005 {cap:false,lang:java} %}
{% endspoiler %}

992 行至 1005 行设置了 RenderSystem 内部的 `shaderGameTime` 值, 以备之后作为 uniform 上传给着色器; 对方块实体和实体渲染派发器进行了一些设置, 暂且略去不谈; 进行了一些光照更新, 这种更新只和游戏机制有关, 也不是我们所关心的.

{% spoiler net/minecraft/client/render/WorldRenderer.java:1012 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 1012 1023 {cap:false,lang:java} %}
{% endspoiler %}

1012 至 1023 行是某些关于调试视锥体的代码. 一般执行的分支是 1017 行的, 对正常的执行流毫无影响.

{% spoiler net/minecraft/client/render/WorldRenderer.java:1025 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 1025 1034 {cap:false,lang:java} %}
{% endspoiler %}

1025 至 1034 行是关于绘制天空和处理背景/雾效的代码. 鉴于正经光影包既不会用到原版天空也不会用到原版雾效, 我们也不关心这些代码的行为, 但作为我们碰到的第一段实际进行世界中物体的渲染的代码, 我们可以简单研究一下 `renderSky` 函数, 来挖掘 ojng 程序员惯用的编码模式.

{% spoiler net/minecraft/client/render/WorldRenderer.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 263 280 {cap:false,lang:java} %}
{% endspoiler %}
以上是 `WorldRenderer` 的构造函数. 考虑到构造函数只执行一次, 而熟悉 OpenGL 的同学可能会认为渲染相关的逻辑需要每帧都执行, 函数体末尾三个 `render*()` 值得解释一下其作用. 以 `renderStars()` 函数为例:

{% spoiler net/minecraft/client/render/WorldRenderer.java %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 577 588 {cap:false,lang:java} %}
{% endspoiler %}

容易看出, 这一函数的实际作用是填充了 `this.starsBuffer` 背后的顶点缓冲的内容. 

通过搜索 `starsBuffer` 的用法, 可以发现正是 `public void renderSky(MatrixStack...)` 函数执行了星星和天空的绘制:
{% spoiler net/minecraft/client/render/WorldRenderer.java:1630 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 1630 1659 {cap:false,lang:java} %}
{% endspoiler %}
这段代码看似只是进行了类似于 `glUseProgram` 的操作, 但实际上同时进行了绘制(`glDrawElements`), 惊不惊喜意不意外? <del>ojng 员工素质展现×3</del> 不仅如此, 在后面的 `getFogColorOverride` 返回非空的东西时[^1], ojng 还当场造了个 buffer, 用 BufferRender 来现场渲染. <del>要复用这种不太变的 buffer 就坚持到底嘛.</del>

此处还出现了一个在源码中很常见的类: BufferBuilder. 参考 `private void renderStars(BufferBuilder buffer)` 函数, 可以得到 BufferBuilder 的主要使用模式:

{% spoiler BufferBuilder usage pattern %}
```java
BufferBuilder bufferBuilder = new BufferBuilder();
VertexBuffer vertexBuffer = new VertexBuffer();

bufferBuilder.begin(VertexFormat.DrawMode.QUADS, VertexFormats.POSITION);
buffer.vertex(x, y, z).next();
...
buffer.end();
RenderSystem.setShader(GameRenderer::getXXXShader);

vertexBuffer.upload(buffer);
vertexBuffer.setShader(MV, P, RenderSystem.getShader())
// or
BufferRenderer.draw(bufferBuilder);
```
{% endspoiler %}

BufferBuilder 类本质上是对 ByteBuffer 的一个包装, 同时也是多个顶点数据和相关绘制参数构成的栈.

+ `bufferBuilder.begin()`可以指定绘制模式和顶点格式, 并将 bufferBuilder 置于准备接受新一个几何体的顶点数据的状态.
+ `bufferBuilder.vertex(x, y, z)` 会将 x,y,z 的值按顺序写入内置的 ByteBuffer, 并将内部状态设置为对下一个元素写入
+ `bufferBuilder.next()` 则会告知 bufferBuilder 完成对当前顶点的写入, 适当调整内部 ByteBuffer 的容量, 并准备好写入下一个顶点.
+ `bufferBuilder.end()` 则结束对当前顶点数据的收集, 根据一定条件补充索引数据(即供 OpenGL 使用的 index buffer), 并将收集到的绘制参数(例如顶点数、绘制模式、顶点格式等)打包, 压入 `bufferBuilder.parameters` 中, 随后重置 bufferBuilder 的状态.
+ `vertexBuffer.upload(bufferBuilder)` 则会弹出栈顶的数据, 并上传到其对应的 OpenGL 顶点缓冲区中.

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

[^1]: 似乎是渲染日出和日落的时候覆盖掉大气雾
