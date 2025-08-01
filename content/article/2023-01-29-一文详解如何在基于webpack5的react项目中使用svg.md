---
title: 一文详解如何在基于webpack5的react项目中使用svg
date: 2023-01-29
tags:
 - webpack
 - svg
categories:
  - 技术
---

本文主要讨论基于webpack5+TypeScript的React项目（cra、craco底层本质都是使用webpack，所以同理）在2023年的今天是如何在项目中使用svg资源的。

<!-- more -->

首先，假定您已经完成基于webpack5+TypeScript的React项目的搭建工作（如果您不太清楚搭建的背景，可以参考这篇笔记：[【个人笔记】2023年搭建基于webpack5与typescript的react项目 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/601013407)）。

# HTML中SVG经典用法

[SVG：可缩放矢量图形 | MDN (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/SVG)

要在一般的html中使用SVG，我们可以直接编写标签：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<div id="app">
    <svg xmlns="http://www.w3.org/2000/svg"
         viewBox="0 0 512 512" width="200" height="200">
        <path d="M256 32C114.6 32 0 125.1 0 240c0 47.6 19.9 91.2 52.9 126.3C38 405.7 7 439.1 6.5 439.5c-6.6 7-8.4 17.2-4.6 26S14.4 480 24 480c61.5 0 110-25.7 139.1-46.3C192 442.8 223.2 448 256 448c141.4 0 256-93.1 256-208S397.4 32 256 32zm0 368c-26.7 0-53.1-4.1-78.4-12.1l-22.7-7.2-19.5 13.8c-14.3 10.1-33.9 21.4-57.5 29 7.3-12.1 14.4-25.7 19.9-40.2l10.6-28.1-20.6-21.8C69.7 314.1 48 282.2 48 240c0-88.2 93.3-160 208-160s208 71.8 208 160-93.3 160-208 160z"/>
    </svg>
</div>
</body>
</html>
```

![010-html-svg-usecase](https://static-res.zhen.wang/images/post/2023-01-29/010-html-svg-usecase.png)

# React编写SVG组件

在React中，React的jsx标签与HTML中的标签几乎是一一对应的，我们可以通过编写jsx来描述组件。所以不难想到，我们可以使用svg以及与其关联的jsx标签（譬如`<path>`、`<g>`等）来手写一个React的SVG组件：

```tsx
export const IconComment = () => {
    return (
        <svg xmlns="http://www.w3.org/2000/svg"
             viewBox="0 0 512 512"
             width="200"
             height="200">
            <path
                d="M256 32C114.6 32 0 125.1 0 240c0 47.6 19.9 91.2 52.9 126.3C38 405.7 7 439.1 6.5 439.5c-6.6 7-8.4 17.2-4.6 26S14.4 480 24 480c61.5 0 110-25.7 139.1-46.3C192 442.8 223.2 448 256 448c141.4 0 256-93.1 256-208S397.4 32 256 32zm0 368c-26.7 0-53.1-4.1-78.4-12.1l-22.7-7.2-19.5 13.8c-14.3 10.1-33.9 21.4-57.5 29 7.3-12.1 14.4-25.7 19.9-40.2l10.6-28.1-20.6-21.8C69.7 314.1 48 282.2 48 240c0-88.2 93.3-160 208-160s208 71.8 208 160-93.3 160-208 160z"/>
        </svg>
    );
}
```

这个IconComment就是一个普通的React组件，编写完成后我们就可以在需要使用的地方引入了：

![020-svg-react-component-usecase](https://static-res.zhen.wang/images/post/2023-01-29/020-svg-react-component-usecase.png)

效果如下：

![030-svg-react-component-display](https://static-res.zhen.wang/images/post/2023-01-29/030-svg-react-component-display.png)

# SVG文件在React中的使用方式

## 组件模式使用

上面我们讲到了如何编写一个svg组件，但一般来说，我们都会让设计出svg资源，然后存放在项目某个目录下并进行使用。我们当然可以把设计出的svg的内容复制到我们的项目中，以组件的方式来使用：

![040-copy-svg-content-to-react-component](https://static-res.zhen.wang/images/post/2023-01-29/040-copy-svg-content-to-react-component.png)

但是每次都需要拷贝一个又一个的组件当然是一件很麻烦的事情，在webpack中我们使用svg资源的时候，其实更希望如同图片资源一样以模块的形式引入（import或者是require）并使用，就像下面一样：

![050-import-svg-component-flow](https://static-res.zhen.wang/images/post/2023-01-29/050-import-svg-component-flow.png)

如果要达到上面的目的，我们首先需要弄清楚一件事情，那就是咱们"import"的"**IconAbc**"到底是个什么？通过上面的代码**反推**，我们很容易回答，IconAbc肯定需要是一个React组件（函数组件或类组件）。

了解webpack的同学都知道，webpack可以通过loader，来处理一个资源在导入的时候会变成什么。但现在在webpack配置中，我们先不添加任何关于svg模块的处理loader，不出意外肯定会报错：

![060-import-svg-without-loader](https://static-res.zhen.wang/images/post/2023-01-29/060-import-svg-without-loader.png)

>ERROR in ./src/icon-comment.svg 1:0
>Module parse failed: Unexpected token (1:0)
>You may need an appropriate loader to handle this file type, currently no loaders are configured to process this file. See https://webpack.js.org/concepts#loaders
>
>译文：您可能需要适当的加载程序（loader）来处理此文件类型，目前没有配置加载程序来处理此文件。请参阅 https://webpack.js.org/concepts#loaders

问题我们已经很清楚了webpack无法找到处理svg模块的loader，那么现在的解决方案是什么呢？我们可以使用`svgr`提供的配合webpack的loader（[Webpack - SVGR (react-svgr.com)](https://react-svgr.com/docs/webpack/)）就可以完成这个任务。

首先安装必要的依赖：`yarn add -D @svgr/webpack`；

然后，配置webpack处理svg文件：

```diff
module.exports = {
    ... ...
    module: {
        rules: [
            {
                test: /\.tsx?/,
                use: [
                    'babel-loader'
                ],
                exclude: /node_moudles/
            },
            ... ...
+           {
+               test: /\.svg$/,
+               use: ['@svgr/webpack']
+           }
        ]
    },
    ... ...
}
```

完成配置以后，重新经过webpack编译打包，运行后会看到控制台的输出：

![070-import-svg-by-svgr](https://static-res.zhen.wang/images/post/2023-01-29/070-import-svg-by-svgr.png)

- 效果1：我们通过console.log输出的IconComment是一个React组件纯函数。
- 效果2：代码中我们使用`<IconComment/>`在屏幕上展示出来了。

> PS：上图中import报错暂时可以不用关心，是IDE类型检查的语法提示，webpack打包是没有问题的，想要深入了解，可以参考：[【长文详解】TypeScript与Babel、webpack的关系以及IDE对TS的类型检查 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/561186916)

回顾整个过程，我们可以用下面的图来描述这个过程：

![080-svgrwebpack-handle-flow](https://static-res.zhen.wang/images/post/2023-01-29/080-svgrwebpack-handle-flow.png)

## 资源模式使用

当然，我们有的时候并不想按照React组件的使用。例如，svg同样可以作为一些元素的背景，这个时候我们需要把svg是为类似于图片一样的资源，就像下面的方式：

![090-use-svg-by-url](https://static-res.zhen.wang/images/post/2023-01-29/090-use-svg-by-url.png)

如果svg的loader配置保持不变，还是`@svgr/webpack`，我们会看到没有起效果，并且，查看对应生成css样式文件，我们可以看到对应的`url('./icon-comment.svg')`被编译为了`url(8ed4ed501566520a5cd0.svg)`：

![100-svg-url-result](https://static-res.zhen.wang/images/post/2023-01-29/100-svg-url-result.png)



这个`8ed4ed501566520a5cd0.svg`是什么呢？可能看起来还有点懵，我们尝试打包编译项目，看一下编译后的产物就知道了：

![110-build-dist-svg](https://static-res.zhen.wang/images/post/2023-01-29/110-build-dist-svg.png)

通过上图的结果可知，很明显svg在这种场景下依然被`@svgr/webpack`这个loader处理为了React组件，又因为咱们是在less/css中引用这个svg，loader内部将这种场景回退到了文件资源存放了。

现在，我们希望webpack在处理这种场景的时候，还是以普通资源的方式进行；同时，在React代码中依然能够将svg资源以组件的形式被引入。好在webpack支持这样的配置：

```js
module.exports = {
    ... ...
    module: {
        rules: [
            ... ...
            {
                // 引用的资源如果是 '${svg-path}/icon-comment.svg?abc'
                test: /\.svg$/,
                resourceQuery: /abc/,
                // 以webpack的资源形式加载（普通资源文件、base64等）
                type: 'asset',
            },
            {
                // 除了上面的匹配规则，我们都按照React组件来使用
                test: /\.svg$/,
                resourceQuery: {not: [/abc/]},
                use: ['@svgr/webpack']
            }
        ]
    },
    ... ...
}
```

>webpack5中的 type: "assets" 是什么？可以看这篇文章：
>
>[资源模块 | webpack 中文文档 (docschina.org)](https://webpack.docschina.org/guides/asset-modules/)

在上述配置中，我们都将匹配svg资源的引用，不同的是，如果这个引用路径带上url query，则使用webpack5的asset资源模块来处理；否则，调用@svgr/webpack来将其转换为React组件。

完成上述的配置以后，我们适当的修改代码，如下所示：

![120-code-show-svg-usecase](https://static-res.zhen.wang/images/post/2023-01-29/120-code-show-svg-usecase.png)

关于关键代码的解释：

1. index.tsx第三行和第四行我们均引入了`./icon-comment.svg`模块，不同的是第四行的引入路径我们还添加了与webpac配置中保持一致的url query = "abc"。同时，在下面我们分别打印了IconComment和IconCommentUrl。
2. 在index.module.less中，`.app`样式中，我们添加的背景也使用`./icon-comment.svg`，也添加了url query = "abc"。

代码运行以后，我们首先从UI上能够看到效果：

![130-svg-usecase-ui-display](https://static-res.zhen.wang/images/post/2023-01-29/130-svg-usecase-ui-display.png)

其次，从控制台也能看到对应的IconComment就是React函数组件；IconComment是svg资源的base64 DataUrl：

![140-svg-usecase-console-output](https://static-res.zhen.wang/images/post/2023-01-29/140-svg-usecase-console-output.png)

# demo地址

本文相关demo已提交至webpack5-react-demo的svg_use_case分支，供读者参考：

[w4ngzhen/webpack5-react-demo at svg_use_case (github.com)](https://github.com/w4ngzhen/webpack5-react-demo/tree/svg_use_case)

