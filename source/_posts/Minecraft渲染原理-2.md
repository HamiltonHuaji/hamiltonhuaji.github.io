---
title: Minecraft渲染原理(2)
date: 2022-09-14 17:15:30
mathjax: true
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
1630 行的 `this.lightSkyBuffer.setShader(...)` 看似只是进行了类似于 `glUseProgram` 的操作, 但实际上同时进行了绘制(`glDrawElements`), 惊不惊喜意不意外? <del>yarn 有的方法命名还是比较有问题的</del> 在后面的 `getFogColorOverride` 返回非空的东西时[^1], ojng 当场造了个 buffer, 用 BufferRender 来现场渲染. <del>要复用这种不太变的 buffer 就坚持到底嘛.</del>

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

BufferBuilder 类本质上是对 ByteBuffer 的一个包装, 同时也是多个顶点数据和相关绘制参数构成的栈.[^2]

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

回到对 `WorldRenderer.render` 的分析上来.
{% spoiler net/minecraft/client/render/WorldRenderer.java:1035 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 1035 1038 {cap:false,lang:java} %}
{% endspoiler %}
1036 和 1038 行的代码是我们啃屎山的开始.

## 屎山: setupTerrain

`WorldRenderer.render` 中的第一座屎山叫 `setupTerrain`. **建议读者直接跳转到本节末尾阅读结论**.

{% spoiler net/minecraft/client/render/WorldRenderer.java:763 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 763 772 {cap:false,lang:java} %}
{% endspoiler %}
`setupTerrain` 函数首先检查了相比上一帧, 摄影机是否移动到了另一个 `ChunkSection`. `ChunkSection` 就是 $16\times 16\times 16$ 大小的那个被一般玩家叫做区块的东西, 但 ojng 叫它 `ChunkSection`. 如果发生了移动, 则调用 `BuiltChunkStorage.updateCameraPosition(x, z)` 来对部分区块设置其 `origin` 成员(仿佛结果是 `x` 和 `z` 的某种阶梯函数):[^3]
{% spoiler net/minecraft/client/render/BuiltChunkStorage.java:68 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/BuiltChunkStorage.java 68 88 {cap:false,lang:java} %}
{% endspoiler %}
随后通过调用 `ChunkBuilder.setCameraPosition()` 告知 `chunkBuilder` 摄像机的新位置. 这是为了帮助 chunkBuilder 进行诸如将区块按离摄像机的距离来排序的操作.

此函数中涉及到了多个未被反混淆的变量, 不妨在此处暂时赋一些名字:

<span style="color:white;background-color:blue;">TODO:</span> <del>我不想编名字了</del>

|   obfuscated |                                                                                                  type |            deobfuscated |
| -----------: | ----------------------------------------------------------------------------------------------------: | ----------------------: |
|  field_34808 |                                                                                    Future<class_6600> | fullUpdateCullingResult |
|  field_34809 |                                                                                         AtomicBoolean |                         |
|  field_34810 |                                                                                               boolean |     shouleUpdateCulling |
|  field_34811 |                                                                                            AtomicLong |     recullScheduledTime |
|  field_34817 |                                                                           AtomicReference<class_6600> |                         |
|  field_34818 |                                                                           WorldRenderer.ChunkInfoList |                         |
|  field_34819 |                                                                              LinkedHashSet<ChunkInfo> |                         |
|   class_6600 |                                class{ChunkInfoList field_34818;LinkedHashSet<ChunkInfo> field_34819;} |                         |
| method_34808 | private void method_34808(LinkedHashSet<ChunkInfo>, ChunkInfoList, Vec3d, Queue<ChunkInfo>, boolean); |                         |
| method_38549 |                                                  private void method_38549(Camera, Queue<ChunkInfo>); |                         |

如果相机相比上一帧移到了不同的 $8\times 8\times 8$ 空间, 或者某个标记 (`WorldRenderer.field_34810`) 被设置时, 则进行一些与区块遮挡剔除相关的操作. 如果没有因为调试而固定使用某个此前捕获的视锥体来进行剔除 (`!hasForcedFrustum`), 且 `Future<class_6600> field_34808` 为空或者不是正在进行的, 则将以下操作提交到 `MAIN_WORKER_EXECUTOR` 线程上执行:

+ 初始化某个广度优先的 FIFO 队列, 即方法 `method_38549` 的内容: 如果当前摄像机所在区块已经被渲染(可能需要换个词, 反正就是 `this.chunks.getRenderedChunk(blockPos)!=null`), 则将当前区块对应的 `ChunkInfo` 加入队列以作为广度优先的开始; 否则将视距内某个高度上所有已渲染的区块对应的 `ChunkInfo` 按从近到远的顺序加入该队列 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 801 802 {cap:false,lang:java} %}
+ 进行某种奇怪的广度优先操作, 即方法 `method_34808` 的内容: 从队列中取出一个 `ChunkInfo`, 将其放入 `field_34808.field_34819`, 并对其 6 个方向上邻接的区块进行某些神必的判断(可能是为了剔除掉例如 6 个方向上都被遮挡的区块), 更新其剔除状态(`updateCullingState`), 并计算其 `propagationLevel` 以便于调试的可视化; 最后在某些情况下将邻接区块放回队列, 并更新 `field_34808.field_34818` 对于该区块的值; 有时还把 `WorldRenderer.field_34810` 设置到 500 毫秒以后, 以规划一次遮挡剔除运算 {% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/WorldRenderer.java 803 804 {cap:false,lang:java} %}
+ 最后将以上结果填充到 `field_34817` 中, 并设置 `field_34809` 以表明有新的剔除结果可用.

提交以上任务后, 渲染线程立即调用 `field_34817.get()`, 以得到此前已有的任务结果. 如果 `builtChunks` 非空, 则从 `builtChunks` 选取 `field_34817.get().field_34818` 中存在的区块加入到另一个队列中, 然后再重复 `method_34808` 的操作.

最后, 在转动视角或有新鲜的剔除结果(field_34809==true)时, 通过调用 `WorldRenderer.applyFrustum()`, 将 field_34817 内所有包围盒与视锥相交或者在视锥体内的区块加入 `WorldRenderer.chunkInfos` 中.

总的来看, `setupTerrain` 函数的主要功能是对区块进行遮挡剔除和视锥剔除; <del>如此复杂的函数笔者不想用心读, 写出来的东西读者也不忍心看, 想必 ojng 的程序员也不想用心维护, 难免有几个 bug .</del>[^4]

## 屎山: updateChunks

上一座屎山 `setupTerrain` 函数过滤过的区块被加入了 `WorldRenderer.chunkInfos`, 因此这座屎山就继续对 `WorldRenderer.chunkInfos` 的内容进行遍历. 只有在某个区块既带有 `needsRebuild` 标记(即满足 `builtChunk.needsRebuild()`)且 `WorldRenderer.world.getChunk(chunkPos.x, chunkPos.z).shouldRenderOnUpdate()` 时, 才会继续对其处理. 对于被处理的区块, 以下行为将发生:

+ 在客户端选项里的 `chunkBuilderMode` 是邻近(`ChunkBuilderMode.NEARBY`)时, 带有 `needsImportantRebuild` 标记或者离摄像机距离小于 $16\sqrt{3}$ 的区块会在渲染线程中立即被同步地重建(rebuild).
+ 在客户端选项里的 `chunkBuilderMode` 是受玩家影响的(`ChunkBuilderMode.PLAYER_AFFECTED`)时, 只有带有 `needsImportantRebuild` 标记的区块会立即被同步重建.
+ 以上的同步重建完成后会清除 `needsRebuild` 标记和 `needsImportantRebuild` 标记, 以避免重建重复发生; 重建完成时一个上传重建结果至显存的任务将被添加到 `WorldRenderer.chunkBuilder` 内部的上传队列 `uploadQueue`
+ 在 `WorldRenderer.chunkInfos` 完成遍历后, `uploadQueue` 将被执行直至清空.
+ 不满足以上重建条件的区块会被收集起来, 丢进 `WorldRenderer.chunkBuilder` 内部的重建队列 `prioritizedTaskQueue` 或 `taskQueue` 里异步重建, 重建完成时上传重建结果的任务同样将被添加到上传队列; 在提交重建任务后, 重建相关标记也会被清除.

区块重建任务最终会调用 `ChunkBuilder.BuiltChunk.RebuildTask.render` 函数, 以收集方块/方块实体的信息, 构建区块的顶点数据, 并写入不同的 `RenderLayer` 的缓冲区中:[^5]
{% spoiler net/minecraft/client/render/chunk/ChunkBuilder.java:576 %}
{% ghcode https://github.com/HamiltonHuaji/minecraft-project-merged-named-sources/blob/master/net/minecraft/client/render/chunk/ChunkBuilder.java 576 637 {cap:false,lang:java} %}
{% endspoiler %}

在完成区块剔除/区块更新和上传顶点缓冲区后, 各 `RenderLayer` 对应的顶点缓冲区就准备好绘制了. `WorldRenderer.renderLayer(RenderLayer, MatrixStack, double, double, double, Matrix4f)` 是具体执行此类绘制的函数. 欲知后事如何, 且听下回分解.

[^1]: 似乎是渲染日出和日落的时候覆盖掉大气雾时使用的网格
[^2]: szszss的[博文](http://blog.hakugyokurou.net/?p=734)中对常常与`BufferBuilder`一起出现的`Tessellator`的来源进行了解释,即试图模拟OpenGL立即模式的惯用法.
[^3]: 剧透一下,`origin`成员会与相机位置进行加减之后作为`chunkOffset`这一uniform变量传递给着色器,并加到顶点的位置上来获得真正的世界坐标.
[^4]: 一般的实践是利用上一帧的深度缓冲,与各区块的包围盒的深度进行比较,bug少不说,程序员写着也舒服.
[^5]: `RenderLayer`可以理解成需要不同的渲染方式的物体构成的集合,例如`SOLID`是完全不透明的方块,`CUTOUT`是有镂空部分的方块,`CUTOUT_MIPPED`是带有mipmapping的前者,`TRANSLUCENT`是半透明的方块.详见[这里](https://neutrino.v2mcdev.com/block/rendertype.html).
