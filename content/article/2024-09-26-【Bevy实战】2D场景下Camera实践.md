---
title: 【Bevy实战】2D场景下Camera实践
date: 2024-09-26
tags: 
 - rust
 - bevy
 - game
categories:
 - 技术
 - 游戏开发
---

[Bevy](https://github.com/bevyengine/bevy)，一个用Rust构建的令人耳目一新的简单数据驱动游戏引擎。如果你是一名Rust开发者，同时又对游戏开发比较感兴趣，那么Bevy一定是你会接触甚至是使用的游戏引擎。当然，本文关注的重点并不是来介绍Bevy，以及它的一些基本概念，关于这块的内容读者完全可以到Bevy的官网、Github主页进行学习；同时，本文也不是一篇介绍Rust编程细节的文章，因此，本文面向的对象是有Rust开发经验（至少是要入门）的小伙伴。接下来就让我们开始本文的主要内容：Bevy在2D场景下的Camera实践。

# 基本概念

Camera摄影机驱动Bevy中的所有渲染。他们负责配置要绘制的内容、如何绘制以及在哪里绘制。为了更形象的讲解，我们先编写如下代码：

> PS：假设读者的其他环境、依赖已经配置Ok

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, startup)
        .run();
}


fn startup(mut commands: Commands, asset_server: Res<AssetServer>) {
    let icon: Handle<Image> = asset_server.load("img1.png");
    commands.spawn(SpriteBundle {
        texture: icon.clone(),
        ..default()
    });
}
```

上述的代码核心过程：

1. 加载Bevy默认的插件
2. 启动阶段（Startup）生成一张精灵图（来自img1.png）

此时我们运行项目，会发现仅有一个没有任何内容的，黑色背景的窗口：

![010](https://static-res.zhen.wang/images/post/2024-09-26/010.png)

造成这个现象的原因是：在默认的情况下，Bevy“游戏世界“中不存在摄像机Camera实体，因此无法将可渲染的内容”拍摄“下来并投射到屏幕上。为了解决这个问题也非常简单，在世界中增加一个摄像机实体即可：

```diff
fn startup(mut commands: Commands, asset_server: Res<AssetServer>) {
+   // add camera.
+   commands.spawn(Camera2dBundle::default());
    // then spawn Sprite.
    let img1: Handle<Image> = asset_server.load("img1.png");
    commands.spawn(SpriteBundle {
        texture: img1.clone(),
        ..default()
    });
}
```

上述代码我们添加了一个2D摄像机的实体，此时再次运行程序可以看到如下的效果：

![020](https://static-res.zhen.wang/images/post/2024-09-26/020.png)

你可以将上面的2D渲染想象成如下的样子：

![030](https://static-res.zhen.wang/images/post/2024-09-26/030.png)

此时，让我们在项目工程中再增加一张图片，并且在之前生成实体的位置再生成一个新的Sprite：

```rust
fn startup(mut commands: Commands, asset_server: Res<AssetServer>) {
    // add camera.
    commands.spawn(Camera2dBundle::default());
    // then spawn Sprite.
    let img1: Handle<Image> = asset_server.load("img1.png");
    commands.spawn(SpriteBundle {
        texture: img1.clone(),
        ..default()
    });
    // add another Sprite with img2.png
    let img2: Handle<Image> = asset_server.load("img2.png");
    commands.spawn(SpriteBundle {
        texture: img2.clone(),
        ..default()
    });
}
```

此时当我们再次运行程序时，可以看到img2.png的内容在img1.png之上：

![040](https://static-res.zhen.wang/images/post/2024-09-26/040.png)

不难理解，先生成的图片实体先渲染，后生成的图片实体后渲染。那么，假设我希望img1.png能够盖住img2.png呢？此时只需要将其位置组件的z轴设置为大于0即可（因为z轴默认是0）：

```rust
fn startup(mut commands: Commands, asset_server: Res<AssetServer>) {
    // add camera.
    commands.spawn(Camera2dBundle::default());
    // then spawn Sprite.
    let img1: Handle<Image> = asset_server.load("img1.png");
    commands.spawn(SpriteBundle {
        texture: img1.clone(),
      	// add transform
        transform: Transform {
            translation: Vec3 {
                z: 1., // <- z > 0
              	x: 100.,
                ..default()
            },
            ..default()
        },
        ..default()
    });
    // add another Sprite with img2.png
		// ...
}
```

> 为了检查效果，笔者还将img1的位置进行一定的偏移。

此时再次运行，可以看到img1已经在img2之上，盖住了img2：

![050](https://static-res.zhen.wang/images/post/2024-09-26/050.png)

同时，通过上面x轴偏移的效果我们也不难知道，在Bevy中的2D的坐标是以**视口**（我们马上介绍什么是视口Viewport）中心为原点的，并非传统GUI的top-left为原点的坐标系统。

注意，视口并非窗口，只是在默认情况下，视口就是整个窗口范围。所以接下来，就让我们开始介绍视口的内容。

# 视口Viewport

通过查阅Camera2dBundle中的Camera组件细节，我们会发现Camera组件存在一个名为`viewport`的字段。根据其文档我们可以知道，这个字段可以用来配置将Camera摄取到的内容渲染到指定`RenderTarget`的矩形区域：

![060](https://static-res.zhen.wang/images/post/2024-09-26/060.png)

所以，在继续对视口相关内容进行介绍前，我们首先要知道上面提到的`RenderTarget`是个什么东西。继续翻阅Camera的代码内容，可以看到Camera还有一个字段`target`：

![070](https://static-res.zhen.wang/images/post/2024-09-26/070.png)

查看`RenderTarget`定义如下：

![080](https://static-res.zhen.wang/images/post/2024-09-26/080.png)

通过其文档我们得知，“target”指的是Camera摄取的内容应该渲染到哪个目标上。已知的有：

1. Window窗体。这是默认的目标。
2. Image图片。也就是说我们有望将游戏世界中的内容摄取并输送到一张图片上。
3. 纹理视图。这个暂时不在本文中进行讲解。

在了解了上述的`RenderTarget`后，我们再回到**视口viewport**的相关内容。此时我们不难推断出，通过配置viewport，我们可以将Camera摄取到的内容输送到相关渲染目标的某个区域（毕竟无论是Window窗体还是Image图像等，它们都是一个平面2D的接收图像的区域罢了）。所以，接下来我们实践一下。将视口配置为一个 400 x 200 的区域：

 ```rust
 fn startup(mut commands: Commands, asset_server: Res<AssetServer>) {
     // add camera.
     commands.spawn(
         Camera2dBundle {
             camera: Camera {
               	// 配置视口的代码
                 viewport: Some(Viewport {
                     physical_position: UVec2::new(50, 50),
                     physical_size: UVec2::new(400, 200),
                     ..default()
                 }),
                 ..default()
             },
             ..default()
         }
     );
     // then spawn Sprite.
 		// ...
     // add another Sprite with img2.png
  		// ...
 }
 ```

上述代码中，我们给对应的Camera配置了视口：在渲染目标（目前就是当前这个窗口）的左上角（50，50）位置开始，宽400，高200的矩形区域。在其余代码保持不变的情况下，运行该程序能够得到如下的效果：

![090](https://static-res.zhen.wang/images/post/2024-09-26/090.png)

此时，我们可以用一张图例来更加形象的表达上述目前的情形：

![100](https://static-res.zhen.wang/images/post/2024-09-26/100.png)

## 关于2D实体的位置等处理

实际上，如果你对上面的内容了解了以后，就很容易知道应该如何处理2D实体的位置、缩放等。一般来说，我们可以有两种方式：

1. 调整世界中的实体的变换
2. 调整摄像机的变换（transform）

假设整个世界中只有一个带有变换组件（Transform）的实体，假设我们希望对这个物体旋转180度，我们即可对这个实体的变换组件进行180旋转变换，也可以对摄像机进行180度的旋转变换。不过，从Bevy这一层次来说，要控制某个实体的位置大小等，最直观的还是对这个实体本身进行变换。

# 多个Camera

一般情况下，我们很容易的产生多个Camera：

![110](https://static-res.zhen.wang/images/post/2024-09-26/110.png)

在前面的代码基础上，我们又Spawn了另一个Camera，这个Camera的视口我们设置到了另一个区域。此时运行程序，我们会发现效果如下：

![120](https://static-res.zhen.wang/images/post/2024-09-26/120.png)

如果你理解了前面的视口流程图，那么这个地方也不难理解：两个摄像机都在“摄取”游戏世界中的2D内容，但由于两个摄像机有不同的视口配置，因此将摄取到的内容渲染到渲染目标上Window上的时候，会将内容渲染到不同的位置：

![130](https://static-res.zhen.wang/images/post/2024-09-26/130.png)

当然，从上图你会发现控制台有这样一句不断打印的警告：

```
WARN bevy_render::camera::camera: Camera order ambiguities detected for active cameras with the following priorities: {(0, Some(Window(NormalizedWindowRef(Entity { index: 0, generation: 1 }))))}. To fix this, ensure there is exactly one Camera entity spawned with a given order for a given RenderTarget. Ambiguities should be resolved because either (1) multiple active cameras were spawned accidentally, which will result in rendering multiple instances of the scene or (2) for cases where multiple active cameras is intentional, ambiguities could result in unpredictable render results.
```

核心意思是，如果场景中存在多个Camera且对应的渲染对象都是一个（比如本例中的同一个Window），需要每一个Camera一个特定的顺序，否则这种歧义可能会造成不可预测的渲染结果（Bevy内部的机制导致的，这里不深究）。所以，为了消除这一警告，我们只需要为不同的Camera设置不同的`order`：

![140](https://static-res.zhen.wang/images/post/2024-09-26/140.png)

## 不同的摄像机渲染不同的实体

有的时候，我们确实需要有多个Camera，且它们各自需要“摄取”并渲染不同的实体。此时，为了实现这个效果，我们需要引入一个组件：RenderLayer。

![150](https://static-res.zhen.wang/images/post/2024-09-26/150.png)

我们将第一个Camera对应的实体和img1对应的实体都添加了`RenderLayer::layer(1)`，标识摄像机将只会“摄取”layer为1的所有实体。同样的，我们针对第二个Camera和img2添加`RenderLayer::layer(2)`：

![160](https://static-res.zhen.wang/images/post/2024-09-26/160.png)

就绪以后，运行程序，我们能看到如下效果：

![170](https://static-res.zhen.wang/images/post/2024-09-26/170.png)

左上角视口的Camera#1，只摄取了img1的内容；而右边的Camera#2则只摄取了img2的内容。

## 关于多Camera的实际应用

上述内容，我们介绍了多Camera的一些效果。那么对于多Camera有什么实际的应用呢？笔者能想到有：

1. ui和游戏世界渲染的分离
2. 某些小地图的呈现

对于第一点来说，最直观的就是，也许我们制作的游戏类似这样的布局：

![180](https://static-res.zhen.wang/images/post/2024-09-26/180.png)

上述是游戏《Caves of Quad》的游戏界面,整体来看，我们就完全可以使用2个以上的Camera来分别渲染左侧的游戏部分和右侧的状态部分。

对于第二点来说，像CS2中的投掷物预览效果，就可以单独用一个Camera来呈现：

![190](https://static-res.zhen.wang/images/post/2024-09-26/190.png)

# 写在最后

本文的代码仓库为：[w4ngzhen/bevy_examples (github.com)](https://github.com/w4ngzhen/bevy_examples)。同时，后续我会添加更多的示例供读者参考。

## 相关阅读以及参考

[Cameras - Unofficial Bevy Cheat Book (bevy-cheatbook.github.io)](https://bevy-cheatbook.github.io/graphics/camera.html)

[bevy/examples at latest · bevyengine/bevy (github.com)](https://github.com/bevyengine/bevy/tree/latest/examples)
