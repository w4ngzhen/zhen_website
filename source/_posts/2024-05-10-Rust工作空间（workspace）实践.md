---
title: Rust工作空间（workspace）实践
date: 2024-05-10
tags:
 - rust
 - workspace
categories:
  - 技术
  - Rust
---

本文将介绍如何使用cargo workspace来管理多个package，并通过实践介绍workspace的一些基础场景下的使用、配置方式。

<!-- more -->

在rust中编写某些中小型项目时，我们通常不会将一个工程拆分为多个package，而是通过一个package下不同的**目录模块**来实现模块拆分，尽管大部分场景下这种开发方式已经足够，然而一旦项目膨胀或是需要遵循模块化的工程设计，我们就不得不将多个模块拆分为独立的package来维护。维护多个package一般有两种方式：1、将多个package拆分为不同的仓库，独立发布crate；2、将多个package存放在同一仓库下，通过cargo workspace来管理，本文主要介绍后者的使用方式。

# 基础配置

假设我们编写了一个rust应用。它分为两个部分：

1. 应用app本身（my_app）。
2. 一个独立的库lib（my_lib）。这个独立的库可能是一个提取出来的工具库，它被my_app项目所依赖。

我们首先创建一个空项目：

```shell
$ mkdir workspace-demo && cd workspace-demo
$ cargo init
```

该命令执行完成后，我们会在当前目录下生成一个名为workspace-demo的目录，并在该目录下生成一个名为Cargo.toml的文件，该文件包含了当前工程的基本信息，包括工程名、版本、依赖等：

![010-basic-project](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/010-basic-project.png)

接着，我们在项目根目录执行如下命令，分别创建两个package：

```shell
$ cargo init my_app --bin
$ cargo init my_lib --lib
```

执行完成以后，cargo帮助我们在项目根目录下创建了两个package：

![020-init-2-packages](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/020-init-2-packages.png)

并且，cargo贴心的帮助我们在项目根目录下的Cargo.toml加入下这段配置：

```diff
+ workspace = { members = ["my_app", "my_lib"] }
```

这段配置意味着，我们刚刚创建的`my_app`和`my_lib`作为了的当前这个项目工作空间的成员（members）。

接下来，让我们删除项目根目录下的src文件夹，然后使用命令（`cargo build`）编译项目下的两个package。

执行命令后会发现一个报错：

![030-no-package-err](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/030-no-package-err.png)

```
Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section must be present
```

这段报错指出：项目根目录下没有`src/lib.rs`，或`src/main.rs`，或者是在Cargo.toml中没有`[[bin]]`、`[[lib]]`字段指定当前根目录下的package。

报错原因在于：首先，项目根目录下的Cargo.toml存在`[package]`字段：

![040-package-field](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/040-package-field.png)

这个字段的存在意味着根目录包含的内容是一个package包，那么这个目录需要符合rust的package结构：目录下存在`src/main.rs`（bin类型包），或存在`src/lib.rs`（lib类型包），或是通过`[[bin]]`、`[[lib]]`字段配置指定该package的入口代码文件。

由于我们已经将`src`目录删除了，也没有额外的配置，所以rust认为我们的目录结构不合法，于是出现上述报错。

所以，为了避免上述的报错，我们将这个`[package]`字段内容移除。

```diff
workspace = { members = ["my_app", "my_lib"] }

- [package]
- name = "workspace-demo"
- version = "0.1.0"
- edition = "2021"

[dependencies]

```

再次进行`cargo build`，会发现新的报错：

![050-dep-section-err](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/050-dep-section-err.png)

```
Caused by:
  this virtual manifest specifies a `dependencies` section, which is not allowed
```

这段报错指出：不允许在虚拟清单类型的工作空间中存在`dependencies`字段。

这里我们需要明白什么是`virtual manifest`？根据[rust圣经提到的](https://course.rs/cargo/reference/workspaces.html#%E8%99%9A%E6%8B%9F%E6%B8%85%E5%8D%95)：

> 若一个 Cargo.toml 有 `[workspace]` 但是没有 `[package]` 部分，则它是虚拟清单类型的工作空间。

这种场景下，我们根目录下的Cargo.toml完全作为整个工作空间下子crate的管理文件，本身并不包含package包。

在本例中，我们希望整个项目下，所有的package都存放到crates目录下，而根目录下不需要放任何的src文件。所以，我们也需要将根目录下的Cargo.toml中的`[dependencies]`字段也一并移除。于是，现在的项目状态如下：

![060-simple-virtual-manifest](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/060-simple-virtual-manifest.png)

最后，我们再次执行`cargo build`，会发现编译成功。


# 子package依赖配置

当然，目前我们仅仅是创建了两个不相干的package。但是实际的场景下，`my_app`会依赖`my_lib`这个crate。为了达到这个目的，我们只需要在`my_app`下的Cargo.toml按照如下方式来定义对`my_lib`的依赖：

![070-workspace-lib-dep-path](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/070-workspace-lib-dep-path.png)

为了让子package依赖到工作空间中其他的package，只需要提供一个文件路径即可，该路径是相对于当前package的路径。

# workspace共享依赖

除了workspace内部之间的依赖以外，我们还可能面临这样的场景：`my_app`和`my_lib`都用到了一个相同的外部依赖库（例如，[serde库](https://docs.rs/serde/latest/serde/)）。为了让这两个库都能依赖到。一种方式是将`my_app`和`my_lib`下的Cargo.toml都按如下方式定义：

![080-simple-dep-serde](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/080-simple-dep-serde.png)

这种方式虽然简单，但是存在一个问题：如果我们将`my_lib`的`serde`升级为一个新的版本，那么我们需要将`my_app`下的`serde`库也升级为新的版本。随着子package的增多，这样的维护方式心智负担会越来越大。那么有没有更优雅的方式呢？答案是肯定的。workspace为我们提供了依赖共享的能力，具体方式如下：

首先，我们在项目根目录下Cargo.toml中增加一个名为`[workspace.dependencies]`的字段，并且在里面定义`serde`的依赖：

```diff
[workspace]
members = ["my_app", "my_lib"]
+ [workspace.dependencies]
+ serde = { version = "1.0.201" }
```

其次，修改`my_app`和`my_lib`下的Cargo.toml的`[dependencies]`字段中关于`serde`库的依赖，改为如下定义方式：

```diff
...
[dependencies]
- serde = { version = "1.0.201" }
+ serde = { workspace = true }
```

整体如下：

![090-workspace-dep](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/090-workspace-dep.png)

按照上述配置以后，`my_app`和`my_lib`不仅都依赖到了`serde`库，而且他们的版本始终保持了一致。如果我们将`serde`升级为一个新的版本，那么`my_app`和`my_lib`都会自动升级。

# workspace还能共享什么？

实际上，除了上述的依赖共享外，还有其他很多的属性可以共享。例如，上述的`my_app`和`my_lib`都是各自在维护自己的版本：

![independent-version](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/100-independent-version.png)

有的场景下，我们希望它们的版本能够保持一致。这个时候，我们同样可以在根目录下的Cargo.toml定义工作空间的版本信息：

```diff
[workspace]
members = ["my_app", "my_lib"]

+ [workspace.package]
+ version = "0.1.0"
+ edition = "2021"
+ license = "MIT OR Apache-2.0"
+ authors = ["w4ngzhen"]

[workspace.dependencies]
serde = { version = "1.0.201" }
```

然后，在各自的package下的Cargo.toml中，将相关字段做如下修改：

```diff
[package]
name = "my_lib"

- version = "0.1.0"
+ version = { workspace = true}
+ # 或
+ # version.workspace = true
+ edition = { workspace = true}
+ authors = {workspace = true}
```

整体如下：

![share-package-config](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2024-05-10/110-share-package-config.png)


# 写在最后

本文简单介绍了rust的cargo workspace的使用方式。当然，本文主要是使用虚拟清单类型（virtual manifest）的工作空间，即，根目录下Cargo.toml不指定任何package。当然，还有一种场景则是：根目录下Cargo.toml可以指定当前目录也是一个package包（通常是bin类型的可执行package），然后将该可执行package依赖的各种二方库通过workspace来配置。本文就不再赘述这块的内容，读者可以自行尝试。

# 参考

http://course.rs/cargo/reference/workspaces.html