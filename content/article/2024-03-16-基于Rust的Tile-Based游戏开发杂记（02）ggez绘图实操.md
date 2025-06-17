---
title: 基于Rust的Tile-Based游戏开发杂记（02）ggez绘图实操
date: 2024-03-16
tags: 
 - rust
 - game-dev
categories:
  - 技术
  - Rust
  - 游戏开发
---

尽管ggez提供了很多相关特性的[demo](https://github.com/ggez/ggez/tree/master/examples)供运行查看，但笔者第一次使用的时候还是有很多疑惑不解。经过仔细阅读demo代码并结合自己的实践，逐步了解了ggez在不同场景下的绘图方式，在此篇文章进行一定的总结，希望能够帮助到使用ggez的读者。

<!-- more -->

# 基本模式

在ggez官方文档中提到一个核心的功能就是基于wgpu图形API的硬件加速的2D渲染：

> Hardware-accelerated 2D rendering built on the wgpu graphics API

ggez的基础绘制模式一般分为3步：

1. 在每一次绘图事件回调中，通过图形上下文构造一个ggez封装的画布Canvas实例；
2. 调用画布的`draw`方法，传入想要绘制的图形（例如一个矩形、一个圆）和相关绘图参数（位置、大小缩放等变换）；
3. 完成所有图像绘制后，调用画布的`finish`方法，向底层图形模块进行一次绘图提交，进而触发底层将最终渲染的图像呈现到画布区域上。

从代码的角度来看，大致如下：

```rust
struct MyState {}

impl EventHandler for MyState {
    fn update(&mut self, _ctx: &mut Context) -> Result<(), GameError> {
        Ok(())
    }

    ///
    /// 绘图
    ///
    fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
        // 1. 构造canvas实例
        let mut canvas =
            graphics::Canvas::from_frame(ctx, graphics::Color::from([1.0, 1.0, 1.0, 1.0]));

        // 2. 绘图
   			// ... ...

        // 3. finish
        canvas.finish(ctx)?;
        Ok(())
    }
}
```

注释中步骤1、3的代码一般来说都很固定，读者根据注释应该很容易理解，这里不再赘述，接下来我们重点关注具体的图形绘制代码。

# 简单绘制一个矩形

当我们希望在窗口上左上角`(10, 20)`的位置绘制一个`40 x 50`的红色矩形时，我们可以通过编写如下的代码来完成：

```rust
    fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
        // 1. 构造canvas实例
        let mut canvas =
            graphics::Canvas::from_frame(ctx, graphics::Color::from([1.0, 1.0, 1.0, 1.0]));

        // 2. 绘图
        let draw_param = DrawParam::new()
      			.color(Color::new(1.0, 0.0, 0.0, 1.0))
            .dest(Point2::from([10., 20.]))
            .scale(Point2::from([40., 50.]));
        canvas.draw(&Quad, draw_param);

        // 3. finish
        canvas.finish(ctx)?;
        Ok(())
    }
```

> 本文将在接下来的内容中逐步介绍不同场景下的绘制，主要会更改关于上述代码中`fn draw`中的内容，其余基本不会改变，所以后续的代码片段没有特殊说明的情况下，均只会贴出`fn draw`中的内容。

我们首先构造一个`DrawParam`实例，通过它来描述我们最终期望绘制的图形的位置和大小。其中，`.color()`不难理解即配置颜色；`dest`指绘制的目标位置；最后，我们定义绘制的矩形的尺寸，但这里值得注意的是，API提供的是`scale`（中文译为“缩放”），并不是一个类似于`size`名称的API，对于初学者来说，这其实是有点反直觉的，别着急，我们稍后就来解释这个地方的概念。

接下来，调用`draw`时，我们第一参数传给的是一个`Quad`实例（的引用），第二个参数就是`DrawParam`数据。这个`Quad`是什么？通过查看源码文档，我们了解到`Quad`是ggez内置的一个最基础的`1 x 1`的Mesh（图形学中一般译为“网格”）：

> A Drawable unit type that maps to a simple 1x1 quad mesh.
> 一种可绘制的单元类型,映射到简单的1x1四方网格。

这里，我们不深究`Quad`这个1 x 1的mesh网格在计算机图形学中的意义，先简单将其理解为一个1 x 1的小方块。那么我们再回看之前提到的`DrawParam::scale`，该API指定的是`Quad`的缩放比例，也就是说，当我们代码中边写的是`scale([40., 50.])`的时候，实际上就是希望将一个原本1 x 1的矩形，使其宽扩大40倍，高扩大50倍。

> 为什么要使用缩放而不是直观的定义尺寸？这涉及到图形学中的变换，我们暂时不在本文中深究。

# 复杂图形

前面的`Quad`读者可以理解为只是ggez内置的一个极为简单的mesh“模板”，通过它我们能在画布指定位置绘制一个指定大小且纯色的矩形块。但实际上，我们在绘图的过程中必然不可能只会画这些简单的方块，或多或少都会画一些不同形状的几何，譬如圆、椭圆、三角形等，以及我们可能还需要为这些几何图形实现渐变，增加边框等效果。作为一款支持2D渲染的游戏框架，这部分的能力当然不会缺失。接下来我们继续介绍ggez在复杂图形的绘图方面的内容。

## Mesh

在ggez中，提供了图形学知识体系中的Mesh数据结构，它是一份包含顶点数据缓存、索引数据缓存，并可以存储在GPU上的数据，并且通过文档我们了解到它的克隆复制成本很低。

> Mesh data stored on the GPU as a vertex and index buffer. Cheap to clone.

关于Mesh的数据结构的含义，如果读者没有学习过计算机图形学，理解起来可能有困难。但在这里，我们可以暂时将它理解为想要通过GPU帮助我们绘图时，提供的一份较为底层的，GPU能直接使用的数据。比如，我们想要画一个矩形，从应用层面的角度，我们可能会定义一个数据结构叫`Rect`，它包含如下的信息：

1. 位置（position）
2. 宽高（width和height）
3. 颜色（color）

但是GPU绘图的时候，我们需要将这些信息转换为GPU能够使用的，更为底层的数据，可能是四个顶点、颜色等数据。

那么，在ggez库中应该如何创建一份Mesh数据呢？以创建一个圆为例，通过阅读文档，我们可以使用`Mesh::new_circle`方法得到：

```rust
let circle_mesh = Mesh::new_circle(
    ctx, // ctx: &mut Context
    Fill(FillOptions::default()), // 填充模式
    [50., 50.], // 圆心
    25., // 半径
    0.01, // 绘制圆弧曲线时多边形长度，越小越圆。
    Color::from_rgb(255, 0, 0) // 颜色
)
```

该方法的入参也非常容易理解，就是一些绘制圆形的基本配置（半径、颜色等）。通过该方法构造一个Mesh后，我们就可以按照之前的方式，通过调用`canvas.draw`方法来绘制它：

```rust
let circle_mesh = Mesh::new_circle(
    ctx,
    Fill(FillOptions::default()),
    [50., 50.],
    25.,
    0.001,
    Color::from_rgb(255, 0, 0)
)?;
let draw_param = DrawParam::default()
    .dest(Point2::from([100., 100.]))
    .scale(Point2::from([1., 1.]))
    .color(Color::new(0.0, 1.0, 0.0, 1.0));
canvas.draw(&circle_mesh, draw_param);
Ok(())
```

看到这段代码，细心的读者会立刻发现，我们已经定义了圆心的位置`[50.0, 50.0]`，但是在构造`DrawParam`数据的时候，又定义了一个：`.dest(Point2::from([100., 100.]))`，即我们希望将图形绘制到`(100, 100)`这个位置，很明显这二者是有冲突的。所以实际是什么结果呢？这里直接给出结论：**图形的最终位置为图形的自身位置 “叠加” `DrawParam`的位置配置**。所以，上述代码中最终圆所处的位置为`(150, 150)`坐标处。

再来讨论`.scale(Point2::from([1., 1.]))`代码的意义。这里我们知道是对图形进行尺寸缩放，在水平和垂直方向上均缩放1.0倍，也就是说不改变图形原有大小。如果我们希望对这个图形在水平方向（x轴）上放大2倍，垂直方向不变，就可以通过scale参数来定制：`.scale(Point2::from([2., 1.]))`。

最后是` .color(Color::new(0.0, 1.0, 0.0, 1.0));`。通过该API，我们定义了图形在绘制的时候使用绿色。很显然，和前面我们构造`circle_mesh`指定的红色（`Color::from_rgb(255, 0, 0)`）是不一致的。**这里最终的结果也是一种叠加，但是它们的叠加不是简单的加减，而是每一单色的值的相乘。**也就是说，按照上面的代码，最终：`Red=255 * 0.0 = 0`，`Green = 0 * 1.0`，`Blue = 0 * 1.0 = 0`，运行以后，你会发现显示出来的是一个黑色圆形！如果你不配置`DrawParam`的`color`，它默认是白色（`[1.0, 1.0, 1.0, 1.0]`），此时，按照相乘的结果，就始终等于你图形定义的颜色了。

下图是一个综合上述讲解后的一个图形：

![010-draw-circle](https://res.zhen.wang/images/post/2024-03-16/010-draw-circle.png)

此外，`DrawParam`还有诸如`rotation（旋转）`、`offset（偏移）`等配置，但是通过阅读底层代码，我们会发现`DrawParam`关于图形位置、缩放等数据核心其实是通过`变换transform`这个字段数据存储的：

```rust
/// DrawParam源码数据结构
pub struct DrawParam {
    /// A portion of the drawable to clip, as a fraction of the whole image.
    /// Defaults to the whole image (\[0.0, 0.0\] to \[1.0, 1.0\]) if omitted.
    pub src: Rect,
    /// Default: white.
    pub color: Color,
    /// Where to put the object.
    pub transform: Transform, // <- 变换是核心
    /// The Z coordinate of the draw.
    pub z: ZIndex,
}
```

至于变换transform，如果学习过图形学、线代、向量等知识理解起来应该完全没有难度。

> DrawParam的其他参数：`pub src: Rect`、`pub z: ZIndex,`我们会在后面实践并解释。

目前为止，我们大致了解了图形绘制的两个部分：1、图形Mesh数据；2、DrawParam绘制定义数据。通过实践我们也了解了它们二者会有定义重叠的部分（例如位置、颜色等）以及叠加的方式。那么，当我们实际开发的时候，面对重叠的部分，究竟是通过配置Mesh本身还是DrawParam呢？要回答这个问题，我们首先要了解一份Mesh数据创建以后，它能做什么。通过阅读文档，我们发现Mesh数据在创建以后，仅仅是提供了一些克隆等API，也就是说，一旦Mesh数据构造完成，就无法对颜色、位置数据进行二次加工设置。而DrawParam数据很容易修改位置、大小、颜色等。也就是说，Mesh数据更偏向于静态绘图，而DrawParam主要负责可变化的绘制。如果在你的场景中，存在对一些图形按照每帧在不同的位置，呈现不同的颜色，那么笔者更建议创建一份图形的Mesh数据，然后在每帧绘制阶段通过临时构造DrawParam来制定当前帧的绘制情况。

举例来说，比如我想在窗体中绘制一个圆形，随着每帧从左到右移动，并且颜色随着从左到右从黑色变成红色：

![020-draw-dynamic-circle](https://res.zhen.wang/images/post/2024-03-16/020-draw-dynamic-circle.gif)

为了达到这样的效果，最直观的做法是我们可以在每一次`fn draw`调用的时候，构造一份对应时刻的对应颜色的圆形的Mesh实例，并进行绘制。但是性能和资源利用更好的方式则是提前创建一份Mesh数据，并在每一次draw调用时，只改变DrawParam的参数即可：

![030-draw-dynamic-circle-code](https://res.zhen.wang/images/post/2024-03-16/030-draw-dynamic-circle-code.png)

## MeshBuilder与MeshData

尽管比起之前的`Qaud`图形，我们现在已经能够绘制圆、三角形、多边形等更多种类的图形，但总的来说依然是一些常见的几何图形，对于实际的应用场景可能还远远不够。比如说，我们希望绘制一座房子，大概像下图这样：

![040-house-draft](https://res.zhen.wang/images/post/2024-03-16/040-house-draft.png)

我们将这个图形分解为三个部分：顶部使用一个棕色三角形作为房顶，房顶下方使用一个黄色矩形作为房屋体，在房屋体内部使用一个棕色的矩形作为门。按照之前的方式，我们首先构造mesh：

![050-multi-mesh-a-house](https://res.zhen.wang/images/post/2024-03-16/050-multi-mesh-a-house.png)

在这段代码中，我们首先在DrawHouseState结构体中增加了3个mesh数据字段：`roof`（屋顶）、`house_body`（房屋体）、`door`（门），在初始化阶段我们构造这三部分并存储起来。

接下来是绘制阶段代码：

```rust
    fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
        // 1. 构造canvas实例
        let mut canvas =
            graphics::Canvas::from_frame(ctx, graphics::Color::from([1.0, 1.0, 1.0, 1.0]));

      	// 2. draw调用了3次！
        let draw_param = DrawParam::default()
            .dest(Point2::from([100., 100.]))
        canvas.draw(&self.roof, draw_param.clone());
        canvas.draw(&self.house_body, draw_param.clone());
        canvas.draw(&self.door, draw_param.clone());

        // 3. finish
        canvas.finish(ctx)?;
        Ok(())
    }
```

在绘制阶段，我们定义了一份DrawParam数据，同时分别对`roof`、`house_body`以及`door`进行绘制。这段代码运行后的效果如下：

![060-house-result1](https://res.zhen.wang/images/post/2024-03-16/060-house-result1.png)

上述代码并不复杂，相信读者能够理解。但是这样的方式并不优雅，因为随着图形结构复杂度愈来越高，我们不可能随时关注一大堆的mesh实例；此外，这样的方式还有一个问题：为了绘制一个“房子”，我们调用了3次`canvas.draw`方法，会有性能上的问题（后续会量化）。

为了解决上述问题，ggez为我们提供了`MeshBuilder`。通过`MeshBuilder`，我们可以将多个mesh同时组合得到一份整体的mesh数据：

![070-single-mesh-a-house](https://res.zhen.wang/images/post/2024-03-16/070-single-mesh-a-house.png)

上面的代码，就是通过`MeshBuilder`依次构造了一个三角形、两个矩形。`MeshBuilder`最后的`build`方法会返回一个`MeshData`，请注意，这的MeshData结构体并不是前面的Mesh数据，而是Mesh结构体创建的来源数据，我们可以将`MeshData`实例传递给`Mesh::from_data`方法来创建Mesh。于是，此处我们只通过一个mesh就包含了整个房屋的图形数据。

最后，在渲染的时候，我们只需要调用`canvas.draw`一次：

```rust
fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
        // 1. 构造canvas实例
        let mut canvas =
            graphics::Canvas::from_frame(ctx, graphics::Color::from([1.0, 1.0, 1.0, 1.0]));

        // 2. DrawParam和绘制一次
        let draw_param = DrawParam::default()
            .dest(Point2::from([100., 100.]));
        canvas.draw(&self.house, draw_param.clone());

        // 3. finish
        canvas.finish(ctx)?;
        Ok(())
    }
```

## InstanceArray

理论上来讲，MeshBuilder提供了将基础图形构成复杂图形以及方便对其进行整体操作的能力。但还有一个场景我们需要进一步讨论：**如何绘制大量的图形？**有的读者可能会说，那好办，在绘图的时候，一个for循环，多次调用`canvas.draw`绘制大量的图形：

![080-draw-house-for400](https://res.zhen.wang/images/post/2024-03-16/080-draw-house-for400.png)

上述的代码，我们通过两个for循环共计400次，依次在`(0, 0)`、`(0, 50)`等位置绘制了50x50的正方形，将原来的房子绘制到对应区域。其中，缩放代码`let scale = [SIZE / 100., SIZE / 100.];`含义是我们的房子本身的尺寸是宽100，高100的尺寸，为了将其刚好会知道50x50的区域内，就需要按照比例缩放：

![090-house-scale](https://res.zhen.wang/images/post/2024-03-16/090-house-scale.png)

上述的代码最终运行的效果如下：

![100-draw-house-for400-result](https://res.zhen.wang/images/post/2024-03-16/100-draw-house-for400-result.png)

从代码逻辑的角度上讲使用for循环还算过得去，但是从性能层面上却有很大的问题。在这里为了可视化性能，我们使用ggez提供的API获得整个应用在运行过程中的fps均值，以此粗略地估算应用在每一次刷新时的性能情况：

```rust
impl EventHandler for DrawMultiHouseState {
    fn update(&mut self, _ctx: &mut Context) -> Result<(), GameError> {
        println!("game fps: {:?}", _ctx.time.fps());
        Ok(())
    }

    fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
				// ... ...
    }
}
```

上述的代码，我们在每一次update中，向控制台打印当前应用的fps值。可以看到在笔者的机器上，未经过编译优化的代码，将这400个小房子绘制到屏幕上，平均的fps在**12**左右：

![110-low-fps](https://res.zhen.wang/images/post/2024-03-16/110-low-fps.png)

对于游戏来说，这么简单的绘制400个图形fps就这么低显然是不应该的。那么这里的最佳实践是什么呢？答案是使用ggez提供的`InstanceArray`。该`InstanceArray`可以用来一次性存储大量的DrawParam数据。当我们要绘制400个房子的时候，实际上只需要构造400个DrawParam，将它们存放到`InstanceArray`中，这400个DrawParam，每一个的`dest`参数都不同，用来表示400个房子的不同位置。当我们需要进行绘制的时候，只需要调用一次`canvas.draw_instanced_mesh`方法，将`InstanceArray`作为第二个参数传入，即可在屏幕上呈现这400个房子，而不是循环400次，每次draw一次：

![120-draw-house-by-instance-arr-code](https://res.zhen.wang/images/post/2024-03-16/120-draw-house-by-instance-arr-code.png)

> 核心本质是每调用一次draw，就是数据从内存到GPU的一次数据传输。

通过使用`InstanceArray`，在同样的编译条件下，在本人60hz刷新率的机器上，绘制这400个图形的fps均值直接拉满60帧：

![130-full-fps](https://res.zhen.wang/images/post/2024-03-16/130-full-fps.png)

# 图片与文本绘制

实际上，图片与文本绘制的模式大体上和前面的图形绘制是保持一致的，都是首先创建一个被绘制的实例：

- 图片：`ggez::graphics::Image`
- 文本：`ggez::graphics::Text`

然后构造`DrawParam`实例或是存放`DrawParam`的`InstanceArray`实例；最后调用`canvas.draw`或`canvas.draw_instanced_mesh`完成单个或批量绘制。接下来我们分别介绍一下ggez绘制图片数据和文本的具体实践。

## 图片绘制

如果是对矮人要塞或是CDDA大灾变等Tile-Based游戏深入了解过，就会发现，这些游戏的图形通常不是一张又一张的小图片存放起来，而是使用一张NxN规格的图片，把所有的图块统一铺在上面的：

![140-tile-img-in-picture](https://res.zhen.wang/images/post/2024-03-16/140-tile-img-in-picture.png)

例如，上图是矮人要塞的Spacefox图块集。你会发现游戏中所有的图形元素都按照16x16的大小统一集中到了这张图片上。那么在实际运行中是如何渲染的呢？游戏只需要将这一张图片加载到内存中，当想要渲染一个“包裹”（上图的第一行倒数第五个就是“包裹”）图形的时候，只需要提供区域偏移信息即可只绘制。

当然，我们先介绍基础图片绘制的方式，将上述一整张图片绘制到窗体上。首先，我们需要加载图片：

```rust
pub struct DrawImageState {
    image: graphics::Image,
}

impl DrawImageState {
    pub fn new(ctx: &mut Context) -> GameResult<Self> {
        /// 使用该路径前，请手动将"spacefox_16x16.png"复制到
        /// 编译的生成的target/debug/resources目录下（没有请手动创建）
        let image = graphics::Image::from_path(ctx, "/spacefox_16x16.png")?;
        Ok(DrawImageState { image })
    }
}
```

上述代码在State结构体中定义了一个image字段，用于存放`ggez::graphics::Image`实例；在初始化代码中，我们通过调用`graphics::Image::from_path`来读取图片`spacefox_16x16.png`。**默认情况下，**图片的搜索目录会从可执行程序所在目录下的`resources`目录中查找。所以为了后续正常运行，我们先暂时手动将图片拷贝至对应目录：

![150-copy-image](https://res.zhen.wang/images/post/2024-03-16/150-copy-image.png)

> 关于ggez中的文件系统，后续会有文章详细讲解。

图片的加载和存储准备好以后，我们在绘图阶段编写如下代码：

```rust
   fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
        // 1. 构造canvas实例
        let mut canvas =
            graphics::Canvas::from_frame(ctx, graphics::Color::from([1.0, 1.0, 1.0, 1.0]));

        // 2. 绘制图片到指定位置
        let dest_point = Vec2::new(0.0, 0.0);
        canvas.draw(&self.image, DrawParam::new().dest(dest_point));

        // 3. finish
        canvas.finish(ctx)?;
        Ok(())
    }
```

在实际运行以后，我们能够看到如下效果：

![160-draw-full-image](https://res.zhen.wang/images/post/2024-03-16/160-draw-full-image.png)

接下来，我们该如何将图片局部绘制到界面上？答案就是使用DrawParam的`src`参数来进行配置。首先，为了绘制上图第一行倒数第5个“包裹”图形，我们首先要确定它处于整张图片的哪个位置。已知图片尺寸为256x256像素，每一个图块尺寸为16x16，“包裹”图块处于水平第12个（基于0索引就是11），垂直第1个（基于0索引就是0）。所以，我们知道“包裹”所在的矩形区域为`x = 11 * 16, y = 0 * 16, w = 16, h = 16`：

![170-tile-rect](https://res.zhen.wang/images/post/2024-03-16/170-tile-rect.png)

于是，我们创建对应区域数据，并作为参数传递给DrawParam：

```rust
    fn draw(&mut self, ctx: &mut Context) -> Result<(), GameError> {
        /// ... ...
      
        // 2. 绘制图片到指定位置
        const TILE_SIZE: f32 = 16.;
        let src_rect = Rect::new(11. * TILE_SIZE, 0. * TILE_SIZE, TILE_SIZE, TILE_SIZE);
        canvas.draw(&self.image, DrawParam::new().src(src_rect).dest(Vec2::new(0.0, 0.0)));
      
 				/// ... ...
    }
```

初看这段代码，应该很好理解，但在实际运行后笔者会发现显示的很有问题。其实，核心原因是ggez中关于`DrawParam::src`所需要的矩形数据是一个相对的数据，它的注释如下：

```rust
#[derive(Debug, Copy, Clone, PartialEq)]
pub struct DrawParam {
    /// A portion of the drawable to clip, as a fraction of the whole image.
    /// Defaults to the whole image (\[0.0, 0.0\] to \[1.0, 1.0\]) if omitted.
    pub src: Rect,
    /// ... ...
}
```

这段注释指的是：传入的Rect矩形的x、y、w、h都是相对于整张图片的相对值，其值范围是0.0到1.0之间的。回到我们的例子，“包裹”图块的对于整张图片的实际位置和尺寸数据是：`x = 11 * 16, y = 0 * 16, w = 16, h = 16`，那么x相对于整张图片是：`(11 * 16) / 水平宽度256`，y相对于图片水平是：`(0 * 16) / 水平高度256`，宽度w相对于整张图是`16 / 256`，高度h相对于整张图是`16 / 256`。所以我们需要做如下的转换处理才能正确绘制：

![180-tile-ratio-rect](https://res.zhen.wang/images/post/2024-03-16/180-tile-ratio-rect.png)

修正代码以后，我们能看到实际的运行效果：

![190-draw-part-image](https://res.zhen.wang/images/post/2024-03-16/190-draw-part-image.png)

## 文本绘制

使用ggez绘制文本，离不开两个重要的结构体：`ggez::graphics::Text`、`ggez::graphics::TextFragment`。其中，`Text`是被绘制的数据，而`TextFragment`主要用于定义一段文本中的局部结构，可以作为`Text`的参数：

![200-draw-text](https://res.zhen.wang/images/post/2024-03-16/200-draw-text.png)

上述的代码，我们首先使用`Text::new("hello, world.")`在画布上绘制文本：`"hello, world."`；然后，我们使用`TextFragment`构建了个两个片段：

1. `TextFragment::new("RED").color(Color::RED)`
2. `TextFragment::new("BLUE").color(Color::BLUE)`

然后通过它们构造了一个新的`Text`实例。这部分的含义是希望绘制的一段文本，`"RED"`使用红色绘制，`"BLUE"`使用蓝色绘制。

上述代码的最终效果如下：

![210-draw-text-display](https://res.zhen.wang/images/post/2024-03-16/210-draw-text-display.png)

# 写在最后

本文主要介绍了使用ggez的图形部分API进行一些基础图形、图片以及文本绘制。尽管ggez在官方提到图形渲染部分是基于`wgpu`的硬件加速的**2D**图形渲染：

> - Hardware-accelerated 2D rendering built on the `wgpu` graphics API

但由于ggez底层使用了`wgpu`，同时也通过一定方式暴露了`wgpu`的相关API，所以实际上我们依然可以进行利用`wgpu`进行3D图形绘制，不过这部分内容需要读者有相关3D图形渲染理论知识以及相关图形库API的使用经验，就不在本文中描述了，笔者可以通过官方[样例代码]([github.com/ggez/ggez/blob/master/examples/cube.rs](https://github.com/ggez/ggez/blob/master/examples/cube.rs))一探究竟：

![220-3d-cube](https://res.zhen.wang/images/post/2024-03-16/220-3d-cube.gif)

本章代码仓库地址：[w4ngzhen/rs-game-dev (github.com)](https://github.com/w4ngzhen/rs-game-dev)

```
cargo run --package chapter_02
```

