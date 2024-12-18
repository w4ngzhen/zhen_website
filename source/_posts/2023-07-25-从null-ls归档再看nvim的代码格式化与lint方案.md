---
title: 从null-ls归档再看nvim的代码格式化与lint方案
date: 2023-07-25
tags:
 - nvim
 - prettier
 - eslint
categories:
  - 技术
  - nvim
---

由于null-lsp的归档和暂停更新，我们需要重新审视并思考还有哪些架构简单易于理解的插件配置方案。本文将介绍脱离null-ls插件体系下的代码格式化和lint的插件配置方案。

<!-- more -->

在之前的文章中《[详解nvim内建LSP体系与基于nvim-cmp的代码补全体系 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/643033884)》中我们提到了null-ls这个插件的目的与作用：诸如prettier、eslint等本身不属于LSP范畴，但又对代码具有解析、处理的外部工具，我们可以通过null-ls插件暴露为语言服务（Language Server），于是用户可以直接使用nvim内置的`vim.lsp.*`相关的API来调用这些prettier、eslint等工具所提供的代码解析、格式化等功能。

然而，null-lsp由于某些原因，即将归档并停止更新了（可以看这里：[IMPORTANT: Archiving null-ls](https://github.com/jose-elias-alvarez/null-ls.nvim/issues/1621)）。在这个背景下，笔者不得不重新审视目前关于代码格式化以及代码lint的插件方案。经过笔者的调研和实践，发现除了null-ls体系以外，其实还有很多架构简单易于理解的配置方案，本文将进行简单的介绍系下的代码格式化和lint配置方案。

# 代码格式化方案

nvim的代码格式化有一个比较经典的插件：[mhartington/formatter.nvim](https://github.com/mhartington/formatter.nvim)。插件安装就不再赘述了，这里主要讲解下formatter这个插件的基本配置思路。

首先，这个插件不会提供格式化代码的能力，它只是一个调用者，你的机器是需要安装相关的代码格式化工具的（譬如要使用prettier，则要通`npm install -g prettier`安装）。

其次，formatter这个插件的思路也很简单：你可以为每一种文件类型（filetype）配置想要调用外部的格式化工具，然后一旦使用插件提供的指令（譬如：`Format`），它就会调用你所配置的外部格式化工具。

![010-formatter-plugin-runtime-arch](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-07-25/010-formatter-plugin-runtime-arch.png)



而这个插件具体的配置方式，通过官方文档我们知道，我们需要在给formatter这个插件进行setup的时候，传入一个`filetype`的字段，这个字段值是一个table，里面的每一个key就是一个filetype，而值则是对应要调用的格式化工具的一段配置。

formatter插件库实际上已经给很多主流的语言都编写了默认的格式化工具调用代码，包括javascript的prettier等。比如，你可以按照如下的方式来配置javascript使用formatter内置编写好的prettier调用代码：

```lua
require("formatter").setup {
  -- ... ...
  filetype = {
    javascipt = { require('formatter.filetypes.javascript').prettier }
  }
  -- ... ...
}
```

那么这个`'formatter.filetypes.javascript'`是什么呢？翻阅formatter插件代码，可以看到路径`lua/formatter/filetypes/javascript.lua`代码：

![020-formatter-repo-filetypes-js](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-07-25/020-formatter-repo-filetypes-js.png)

而该代码中的`local defaults = require "formatter.default"`就来源于`lua/formatter/defaults`目录下的模块，像prettier就来自于对应文件preitter.lua：

![030-formatter-repo-default-prettier](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-07-25/030-formatter-repo-default-prettier.png)

这里也能很清晰的看到，formatter调用prettier的时候，就是调用的命令行环境中的`prettier`，所以我们才在一开始的时候提到，需要安装对应外部格式化工具，并且能在命令行形式被访问调用。

那么，再次回到笔者自己的配置：

![035-my-formatter-plugin-setup](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-07-25/035-my-formatter-plugin-setup.png)

主要分为了两个部分：

1. 针对javascript、typescript以及它们的react扩展（jsx、tsx）文件类型，我们都配置了对应的格式化器使用prettier；这里通过lua脚本for遍历来方便的为一系列的文件类型均使用了prettier。

2. keymap按键映射。使用leader+大小写f键，来映射调用formatter插件提供的FormatWrite和Format指令。

需要注意的是，这里的格式化要和nvim的lsp格式化（`vim.lsp.buf.format()`）区别开来。formatter插件的格式化，主要是使用外部格式化工具进行，往往更加专注代码格式化本身；而lsp的格式化是通过语言服务（往往伴随更加复杂的代码分析）完成的。它们的各有各的侧重点，但在笔者看来，如果一门语言有更加专业的格式化方案（譬如本例中js使用prettier这种成熟方案），那么笔者建议使用formatter插件配合对应的专业格式化工具来完成代码格式化，而之前文章《[详解nvim内建LSP体系与基于nvim-cmp的代码补全体系 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/643033884)》中提到的监听`LspAttach`事件，然后注册keymap映射`<cmd>lua vim.lsp.buf.format()<CR>`可以用来兜底哪些暂时不使用专门的格式化工具的场景：

![040-LspAttach-format](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-07-25/040-LspAttach-format.png)

于是，在笔者的配置下，如果本人正打开的是有prettier规范的前端项目的时候，总是会使用`<leader>+f/F`来调用prettier来进行代码格式化；而假设正在编辑一段lua代码，那么会使用ctrl+alt+L来通过lua的语言服务进行代码格式化。读者在理解了这两种格式化机制以后，自行涉及方案了。

# lint方案

lint方案和上面的格式化会有所差别。在不使用null-ls的情况下，lint方案实际上完全可以通过nvim自己的lsp模块配置外部工具完成。翻阅lspconfig目前已经支持的语言服务，会看到eslint也在其中，同时你也能看到很多其他语言的lint都在这个语言服务的说明文件里面。

![050-lint-eslint-by-ls](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-07-25/050-lint-eslint-by-ls.png)

也就是说，至少对于lspconfig插件，它将各种lint也都视为了语言服务（至于格式化为什么没有作为语言服务，个人觉得格式化的功能比较单一）。同样的，我们只需要安装lint外部的命令行工具或者已经包含了lint功能的语言服务工具（说到底还是要在机器上安装对应的命令行工具），就能够获得lint的能力了。 

比如，在笔者的机器上为了使用eslint功能，则会全局安装vscode-langservers-extracted，里面就包含了一个名为`vscode-eslint-language-server`的专门做ESlint的语言服务能够在命令行中访问。然后只需要像配置普通的语言服务一样来启动eslint：

```lua
require("lspconfig").eslint.setup({})
```

# 总结

总的来说，在没有null-ls这套体系的参与下，我们同样也能够很方便的配置格式化和lint。

先说代码格式化，在nvim中，格式化有两种形式，一种是调用外部**独立专用**的格式化工具来完成代码格式化；另一种就是通过nvim提供的lsp模块的format来进行格式化，从本质上来讲，后者和前者是一样的，毕竟语言服务不过也是一种特殊的外部工具而已；

再讲nvim工程调用lint工具，这里lspconfig讲lint工具也视为了一种特殊的语言服务，因为lint就支持`diagnostics`、`code actions`等。所以，实际上只需要安装了对应的lint工具（或是包含了lint功能的语言服务），然后通过lspconfig就能很方便的启用了。

> PS：笔者已经将自己的nvim配置中的null-ls和需要基于null-ls的prettier.nvim、eslint.nvim都删除了；换成使用formatter.nvim和lspconfig启用eslint来分别替代代码格式化和eslint检查了。
