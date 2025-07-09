---
title: 基于Rust的Tile-Based游戏开发杂记（01）导入
date: 2024-02-27
tags:
 - game-dev
 - tile-based
 - rust
categories:
  - 技术
  - Rust
  - 游戏开发
---

关于使用Rust开发Tile-Based游戏系列文章。

<!-- more -->

# 什么是Tile-Based游戏？

Tile-based游戏是一种使用tile（译为：瓦片，瓷砖）作为基本构建单位来设计游戏关卡、地图或其他视觉元素的游戏类型。在这样的游戏中，游戏世界的背景、地形、环境等都是由一系列预先定义好的小图片（即tiles）拼接而成的网格状结构。每个tile通常代表一个固定的尺寸区域，可以是一块地表、一片水域、一堵墙、一棵树或者是任何构成游戏环境的一部分。

设计者通过组合不同的tile，能够快速创造出多样化的游戏地图，这在许多类型的游戏如角色扮演游戏（RPG）、策略游戏、冒险游戏、解谜游戏以及一些独立游戏和roguelike游戏中尤为常见。Tile-Based游戏和ASCII游戏有一定的关联，比如经典的CDDA大灾变，无论是是使用ascii字符进行渲染，还是使用图片进行渲染，对应坐标位置上的元素总是一致的：

![010-ascii-tile-based-game](https://static-res.zhen.wang/images/post/2024-02-27/010-ascii-tile-based-game.png)

那么从技术层面出发，想要开发出类似这样的游戏，我们需要准备哪些东西呢？

# 游戏框架选择

一个基本的游戏循环逻辑如下：

1. 游戏运行后我们需要使用一块画布来承载游戏应用，它能够接收用户的各种输入，包括不限于键盘、鼠标等输入事件，以便我们在后续的游戏逻辑更新环节响应外部输入；
2. 需要一个模块来承载游戏逻辑，通过接收到外部输入，来更新游戏状态；
3. 需要一个渲染模块来扮演图形输出的能力，无论是渲染ascii还是各种2D、3D图形，都需要通过这个模块来完成。

伪代码如下：

```
0. 游戏初始化（通常是系统层面初始化，而不是游戏逻辑初始化，窗体准备）
1. 资源加载，游戏准备
2. 游戏循环处理
while(游戏进行中) {
    2-1. 处理各种输入
    2-2. 更新游戏各种状态数据
    2-3. 图形渲染
}
3. 游戏退出，资源释放
```

在Rust生态中已经有很多较为成熟且流行的库来帮助封装对系统底层的调用，让我们更加关心核心的游戏循环处理。比如，`winit`库封装了不同操作系统平台关于窗体创建的底层实现，暴露出safe的Rust层面API，我们可以直接使用。当然，这个库仅仅提供原生窗体以及相关事件响应封装，并不提供内容渲染能力，换句话说，当我希望在游戏中能够渲染一些文字，或者一些图片，或者一些动画，这些需要对于`winit`**本身**来说是无法做到的，需要配合一些另外渲染库通过底层的图形API来完成渲染并绘制到`winit`创建的窗体上。

当然，已有一些游戏框架、游戏引擎层面的库供我们使用，比较常见有：`Bevy`、`ggez`、`Piston`等等。它们都提供了基本窗体控制、事件循环处理以及对图形API的更上层封装，对于开发者来说，使用这些库以后，代码绘制图形也无需过多的关心底层的图形绘制细节以及各种事件处理。

笔者在对上述游戏开发框架、引擎库进行调研并经过**多次实践**以后，决定使用[ggez](https://github.com/ggez/ggez)这个**2D游戏框架**来完成本系列文章的游戏开发。

这个库提供了如下的功能：

1. 文件系统抽象，允许您从文件夹或 zip 文件加载资源
2. 基于`wgpu`库图形API构建的硬件加速2D渲染
3. 通过`rodio`库加载和播放.ogg、.wav和.flac文件
4. 使用`glyph_brush`库进行TTF字体渲染
5. 通过回调轻松处理键盘和鼠标事件的界面
6. 用于定义引擎和游戏设置的配置文件，简单的定时和FPS测量功能
7. 集成了`mint`库作为数学库
8. 一些更高级的图形选项：着色器、实例化绘图和渲染目标

除开上述一些能力外，`ggez`还有一个更加吸引笔者的地方是，相比于`Bevy`、`Piston`等库来说，它更加轻量级，更加适合入门游戏开发者，其仓库代码量比起`Bevy`和`Piston`来说也更少，如果想要更深入的了解`ggez`这个库，代码也更易阅读。

最后需要注意的是，`ggez`只能说一款**仅支持2D游戏开发**的框架而不算游戏引擎，并没有集成物理引擎、GUI、AI、ECS框架以及网络通讯等能力，但是对于开发Tile-Based游戏来说，`ggez`本身已经足够了。

## 实践ggez

通过官方文档，我们可以快速的搭建并运行一个窗体，同时在窗体中渲染一些内容：

```rust
use ggez::{Context, ContextBuilder, GameResult};
use ggez::graphics::{self, Color, DrawParam, Quad};
use ggez::event::{self, EventHandler};
use ggez::mint::Point2;

fn main() {
    let (ctx, event_loop) =
        ContextBuilder::new("my_game", "Cool Game Author")
            .build()
            .expect("error");
    let my_game = MyGame {
        x: 0,
        to_right: true,
    };
    event::run(ctx, event_loop, my_game);
}

struct MyGame {
    x: i32,
    to_right: bool,
}

impl EventHandler for MyGame {
    fn update(&mut self, _ctx: &mut Context) -> GameResult {
        // 更新方块状态
      	// ...
        Ok(())
    }

    fn draw(&mut self, ctx: &mut Context) -> GameResult {
      	let mut canvas = graphics::Canvas::from_frame(ctx, Color::WHITE);
        // 方块绘图
      	// ...
        canvas.finish(ctx) // 提交绘图更新
    }
}
```

按照官方文档，我们编写上述示例代码，运行以后会看到如下效果：

![020-ggez-demo](https://static-res.zhen.wang/images/post/2024-02-27/020-ggez-demo.gif)

> 具体代码仓库会在文章末尾给出

# GUI库选择

## 介绍

游戏中并非一定要有GUI，比如一些小游戏可能一个回车就开始了游戏，或是只需要通过一些绘图API“画”一个按钮，而不需要一些复杂GUI控件，但考虑到后期，我们的Tile-Based游戏支持处理更加复杂的用户输入，我们不可能完全通过代码“画”一些按钮、表格，或者其他的GUI控件，所以考虑还是引入GUI库来支持渲染一些常用的GUI控件（按钮、文本框、滑动条等）。

游戏开发中的GUI库不同于一些普通桌面应用，因为通常来讲游戏是按照GameLoop一帧一帧不断渲染的，这种GUI渲染模式通常叫做立即模式（immediate mode），即每一帧都在“画”一个控件。更详细的介绍，可以参考下面这些文章：

- [立即模式GUI和保持模式GUI](https://zhuanlan.zhihu.com/p/534695668)
- [why-immediate-mode](https://github.com/emilk/egui#why-immediate-mode)

> 此外，立即模式实现动画过渡效果非常容易，因为立即模式是每一帧都绘制一遍，我们可以通过每一帧时间间隔和上一帧的状态来计算出当前帧的状态，进而实现一种平滑过渡的效果。

在Rust中，同样有很多库可以帮助我们完成GUI的渲染，例如：`egui`、`iced`等等。其中，egui算是比较主流的立即模式GUI库了，读者可以访问egui的Example网站看到效果：[web demo](https://www.egui.rs/)。当然，egui本身可以支持很多渲染后端（backend），即egui本身只提供渲染的计算（比如这里要画一个按钮，那里要画一个表格），而具体的渲染工作交给后端来完成，这个后端可以是OpenGL、Vulkan、DirectX以及wgpu等。

在前面，我们选择了ggez游戏开发框架，它本身具备渲染能力（底层使用wgpu），所以我们可以将ggez的wgpu作为后端，将egui与ggez结合，当然，已经有开发者开发好了：[ggegui](https://github.com/NemuiSen/ggegui)，我们只需使用即可。

## 实践egui（ggegui）

在原有ggez代码的基础上，我们添加ggegui库，并增加相关的代码后，能够看到如下效果：

![](https://static-res.zhen.wang/images/post/2024-02-27/030-ggez-with-egui.gif)

> 由于GIF录制原因导致不清晰，读者可以自行运行Demo代码查看效果

# ECS框架选择

## 介绍

什么是ECS架构？这里有一篇写的很好的文章来介绍ECS架构：[游戏开发中的ECS 架构概述](https://zhuanlan.zhihu.com/p/30538626)。

简单来讲，ECS架构的核心思想是将游戏中的所有事物抽象为一个个独立的实体（Entity），每个实体都拥有自己的组件（Component），组件是游戏状态的最小单元，它可以具备一些数据。而系统System则是对游戏状态的更新处理，它可以根据游戏状态的变化来更新游戏状态，修改这些组件中的数据。值的注意的是，系统从来不会拥有任何的实体或组件，系统仅仅无状态的逻辑处理。

对于我们的Tile-Based游戏来说，就十分适合基于ECS架构来处理游戏循环中的状态更新。所以在后续的开发中，我们会引入一套ECS框架来处理游戏中关于状态更新的部分。

在笔者调研并实践了`Bevy_ecs`、`specs`等ECS框架库以后，决定使用[specs](https://github.com/amethyst/specs)库。未使用`Bevy_ecs`是考虑到：

1. `Bevy_ecs`尽管用户更多，但是它本身模块不太方便与**非**Bevy生态结合（PS：Bevy是一整套游戏引擎，包含了游戏引擎本身、ECS框架、GUI库等等）；
2. `specs`这个库是一个独立的ECS框架库，主打轻量级高性能，API清晰独立，且易于与各种逻辑进行结合。我们可以在自己的逻辑代码中更加方便的控制ECS中的逻辑处理。

## 实践specs

specs的代码逻辑细节就不在本文中进行展示了，烦请读者自行阅读specs的[官方文档](https://specs.amethyst.rs/docs/tutorials/01_intro)，以及笔者的仓库代码进行学习。

![040-ggez-egui-specs](https://static-res.zhen.wang/images/post/2024-02-27/040-ggez-egui-specs.gif)

# 最后

无论是[CDDA大灾变](https://cataclysmdda.org/)、[矮人要塞](https://www.bay12games.com/dwarves/)，还是[rogue](https://github.com/Davidslv/rogue)、[nethack](https://github.com/NetHack/NetHack)等游戏，它们都有着非常丰富的游戏内容，而这些内容除开本身有趣的逻辑外，往往都需要通过一些算法来实现，比如：地图生成、寻路、AI、战斗等等。后续内容笔者将会针对不同的游戏模块内容，介绍相关的开发逻辑、算法。比如经典的a-star寻路算法，基于simplex噪声、perlin噪声的地图生成算法，以及基于ECS的战斗系统等等。另外，笔者也会介绍使用ggez框架过程中学习到的绘图一些经验、技巧，尽情期待！

代码仓库：[w4ngzhen/rs-game-dev (github.com)](https://github.com/w4ngzhen/rs-game-dev)
