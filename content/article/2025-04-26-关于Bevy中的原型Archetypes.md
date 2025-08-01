---
title: 关于Bevy中的原型Archetypes
date: 2025-04-26
tags: 
 - Rust
 - Bevy
categories: 
 - 游戏开发
---

# 认识Bevy中的原型

Bevy是基于ECS（Entity-Component-System）架构的游戏引擎，其中的**Entity实体**是游戏中的一个基本对象，但实体本身通常只是一个标识id，它不包含任何具体的数据或行为，**只是组件（Component）的容器**。

具体一点，要在Bevy（或是绝大多数基于ECS架构的游戏引擎）的游戏世界中去创建一个实体，其做法通常是创建一个包含一组组件的合集，该组件合集从概念上来表达某种游戏实体。

例如，创建一个包含生命值（Health）、位置（Position）的敌人角色，我们并不会定义一个名为Enemy的类型去继承某些对象，为其添加生命值数据和位置数据，而是定义3个组件：Enemy、Health以及Position，他们都是组件，当我们同时生成一个包含这3个组件的合集的时候，从概念上它就成为了一个包含生命值以及位置的敌人角色实体：

![010](https://static-res.zhen.wang/images/post/2025-04-26/010.png)

针对上述实体的表现形式，从底层存储的角度来看，我们可以先假设通过**表格table**的每一行记录来表达实体：

![020](https://static-res.zhen.wang/images/post/2025-04-26/020.png)

> 上图表示游戏世界中的两个敌人实体，每个实体包含独立的`Position`和`Health`组件数据。

接下来让我们加入一个“玩家”角色，我们首先需要新增一个Player组件，然后在游戏世界中创建包含Player、Postion以及Health三个组件的合集，该实体从概念上就视为一个“玩家”角色，同时按照之前表格的数据表现形式，我们需要给对应的表格再增加一列Player，游戏世界对应的实体情况如下：


![030](https://static-res.zhen.wang/images/post/2025-04-26/030.png)

我们可以很显然想到上述table存在一个问题：随着后续实体增多，这些实体包含的组件千变万化，会造成这个table表格每一行记录是不连续的，且列会越来越多。

![040](https://static-res.zhen.wang/images/post/2025-04-26/040.png)

此时如果我们想写向量化的代码来算法优化处理这些数组记录，那么上述结构中不连续处的空值将会影响其效果。

> 向量化编程意味着一次对整个数组或数据矢量进行操作的代码，而不是顺序处理单个元素。
> 
> 向量化操作可以使用像[SIMD（单指令流多数据流）](https://zh.wikipedia.org/wiki/%E5%8D%95%E6%8C%87%E4%BB%A4%E6%B5%81%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%B5%81)这样的硬件优化来有效地执行。这些指令允许在多个数据元素上同时执行相同的操作，通常会提高性能。

为了解决上述的数组元素不连续的问题，Bevy将包含不同组件的实体拆分到不同的记录表中：

![050](https://static-res.zhen.wang/images/post/2025-04-26/050.png)

可以看到，原本包含所有实体记录的单个table拆分为**两种**table，同时其每一行记录是连续的。在Bevy中，会将这两种表（组件类型构成）视为两种**原型（Archetype）**。

## 拆分为多个原型的优势

读者可能会好奇这样做拆分的目的是什么。实际上，拆分目的是能够充分利用并行计算，在Bevy内部同时处理不同的系统。假设现在有如下两个系统（system）：

```rust
fn handle_player_position(
  player_positions: Query<&mut Position, With<Player>>,
) {
  // ...
}

fn handle_not_player_position(
  not_player_positions: Query<&mut Position, Without<Player>>
) {
  // ...
}
```

从系统1`handle_player_position`的query参数我们可以知道，运行该系统时，我们会处理**具有**Player组件的实体的位置数据，而从系统2`handle_not_player_position`的query我们知道在运行该系统时，会处理**不具有**Player组件的实体的位置数据。

对于拆分的table，我们可以很容易的完成并行计算，因为系统1只会查询table1的记录，而系统2则肯定只会查询table2的记录，而不会查询到table1中：

![060](https://static-res.zhen.wang/images/post/2025-04-26/060.png)

## 关于原型的创建

当我们调用Bevy提供的API来构建实体的时候，Bevy就会根据此时所传入的组件列表来**查找并使用**或**新建**的一个原型。即，如果找到了满足当前组件列表定义的原型时，就直接使用该已存在的原型，此时只需要把这一组组件的一些上下文（特别是id关系等）记录到原型中；如果没有找到满足的原型，则新建一个原型实例，同样把上述的相关上下文保存到该新建的原型中，并在后续使用。

> Bevy通过惰性初始化边的映射关系，仅在首次遇到组件变更时创建新原型并记录边，后续直接复用。

![070](https://static-res.zhen.wang/images/post/2025-04-26/070.png)

## 关于原型的更新

注意，这里说的是原型的更新，而非原型下面的某个组件数据的更新，所谓原型的更新发生在其对应实体内组件的增删。比如，对于某个实体具有一个名为`Visibilty`的组件来标记一个实体是否能被看到，我们想要隐藏该实体时，只需要将`Visibility`组件从这个实体中移除即可：

![080](https://static-res.zhen.wang/images/post/2025-04-26/080.png)

注意，此时该实体id并不会发生变化，只是该实体所包含的组件少了`Visibility`组件，进而导致该实体所对应的原型发生了变化：

![090](https://static-res.zhen.wang/images/post/2025-04-26/090.png)

上图中，实体原本属于原型`(Enemy, Position, Health, Visibility)`，移除`Visibility`组件后，会被迁移到原型`(Enemy, Position, Health)`，这个过程会引发上述原型创建的流程，选择已有的原型或建立新的原型。

当然，读者可能会有这样的疑惑，当我们频繁的添加或移除某个实体内的组件时，原型会被频繁的创建，实体信息会被频繁地移动到各个原型中，这其中的性能如何保证呢。对于这个问题，Bevy采取了操作“图”化的设计，即每一个原型实例在其内部会存储一组信息，该信息包含用来记录这个原型一旦发生相关的改变，能够通向的下一个原型的id。

以上面移除`Visibility`组件为例，改变前的原型`(Enemy, Position, Health, Visibility)`我们假设称其为`AT1`，当我们**首次**将某个实体的`Visibility`组件移除时，由于此刻运行时内部没有其他的原型，所以Bevy会创建一个新的原型`(Enemy, Position, Health)`（我们假设称其为`AT2`）；之后，Bevy会生成这样一条上下文信息：“移除`Visibility`时，指向`AT2`”，将其存储到`AT1`中，这条上下文信息在Bevy内部实现被定义为一条**边（Edge）**。如此一来，在后续再次出现同样原型的“敌人角色”实体移除`Visibility`的时候，可以通过`AT1`中的存储的“移除`Visibility`时，指向`AT2`”来快速索引到`AT2`中。

![100](https://static-res.zhen.wang/images/post/2025-04-26/100.png)

> 类似状态机的设计

# 写在最后

本文是近期阅读Bevy相关的资料和源码时候的一些所想所感，思来想去最终整理并写下了这篇文章，当然这其中可能有些错误或者不准确的地方，还望读者见谅。
