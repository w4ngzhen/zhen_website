---
title: 掌握webpack（一）一张图让你明白webpack中output的filename、path、publicPath与主流插件的关系
date: 2022-12-31
tags:
 - webpack
categories:
  - 技术
---

webpack的核心概念，放到2022年相信很多的小伙伴都已经非常清楚了。但是，对于webpack配置中的output.path、output.filename以及output.publicPath，还有很多小伙伴还不理解。本文讲围绕output.filename、output.path与output.publicPath，讲解它们的功能，并分析这些配置与webpack中常使用到的MiniCssExtractPlugin、HtmlWebpackPlugin等插件的关系。

<!-- more -->

直接上总结图

![160-v3-path-filename-publicPath](https://static-res.zhen.wang/images/post/2022-12-31/160-v3-path-filename-publicPath.png)

# 基础环境搭建

我们现在基于webpack搭建了一个前端项目，完成项目初始化，并安装webpack三件套：

```shell
yarn init
yarn add -D webpack webpack-cli webpack-dev-server
```

安装完成以后，我们在项目根目录下创建一个webpack.config.js，一个极简的配置如下：

```js
const {resolve} = require('path');
module.exports = {
    mode: 'development',
    entry: {
        main: resolve(__dirname, 'src', 'index.js')
    },
    output: {
        filename: 'main.js',
        path: resolve(__dirname, 'dist')
    }
}
```

然后，在package.json中添加脚本：

```diff
+ "scripts": {
+   "build": "webpack --config webpack.config.js"
+ },
  "devDependencies": {
    "webpack": "^5.75.0",
    "webpack-cli": "^5.0.1",
    "webpack-dev-server": "^4.11.1"
  }
```

接着，我们在src目录下创建一个index.js文件，这个js文件的内容就是从dom上找到id为app的元素，并给其内部添加一个文本`"hello, world"`：

```js
document.getElementById('app').innerText = 'hello, world'
```

最后，我们运行webpack的构建过程：

```
yarn build
```

运行以后，就会在项目根目录下的dist目录下生成main.js。

![010-base-config](https://static-res.zhen.wang/images/post/2022-12-31/010-base-config.png)

*注意：这里并没有配置关于js的解析，因为webpack默认就会处理js文件。*

## 引入HtmlWebpackPlugin

仅仅是生成目标js文件，可能还不是我们期望的效果。对于一个项目来说，我们通常还希望有一个html来展示UI，并运行js代码，但是手工创建可能不能是一个好的方案。这里，我们引入本项目的第一个插件：**HtmlWebpackPlugin**。

```
yarn add -D html-webpack-plugin
```

HtmlWebpackPlugin插件**基础**功能：

1. 它会使用一个模板来生成一个html；
2. 在生成的html中插入节点（譬如，js对应的script节点等）。

安装好该插件以后，在之前的webpack配置中，我们适当的修改：

- 引用插件，并new一个HtmlWebpackPlugin实例（不添加其他配置）

```diff
 const {resolve} = require('path');
+const HtmlWebpackPlugin = require('html-webpack-plugin');
 module.exports = {
     mode: 'development',
     entry: {
@@ -7,5 +8,10 @@ module.exports = {
     output: {
         filename: 'main.js',
         path: resolve(__dirname, 'dist')
-    }
+    },
+    plugins: [
+        new HtmlWebpackPlugin({
+
+        })
+    ]
 }

```

让我们再次运行构建脚本后，我们会发现，dist目录中，不仅仅生成了main.js，还生成一个index.html：

![020-a-new-indexhtml](https://static-res.zhen.wang/images/post/2022-12-31/020-a-new-indexhtml.png)

通过检查这个index.html的内容可以看到，这个插件不仅仅帮我们生成了一个html，还在这个html中的head节点中创建了一个script节点，并且src属性填写的是main.js。

![030-check-new-indexhtml](https://static-res.zhen.wang/images/post/2022-12-31/030-check-new-indexhtml.png)



此时，我们使用浏览器直接打开这个index.html，尽管是在文件系统，但浏览器还是可以通过script节点中的属性`src="main.js"，从index.html所在同级目录中加载main.js。然而，运行起来有报错：

![040-raw-indexhtml-script](https://static-res.zhen.wang/images/post/2022-12-31/040-raw-indexhtml-script.png)

> PS：这里有同学可能会认为是script节点在body以前加载的，所以会报错。但是实际不是这样的，这里script节点中有一个`defer`属性，这个属性表明，文档加载完毕以后才会执行main.js（[MDN - defer](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/script#attr-defer)），所以，我们不用担心由于DOM未加载完就执行js代码而造成报错。

**这个地方的问题在于：我们的main.js中会执行查找id为app的元素，但是实际生成的html是没有这个元素的。**

为了解决上述的问题，我们希望能够自定义生成index.html。通常的做法就是：

1. 在项目根目录创建一个public目录，在其中创建一个index.html（项目根目录/public/index.html），内容如下（重点是body里面添加了`<div id="app"></div>`）：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>这是一个模板HTML</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <div id="app">
  </div>
</body>
</html>
```

2. 然后，修改webpack配置中，关于HtmlWebpackPlugin的配置，配置插件`template`参数，表明使用上述的创建的index.html：

```diff
     plugins: [
         new HtmlWebpackPlugin({
+            template: resolve(__dirname, 'public', 'index.html')
         })
     ]

```

我们再次运行构建，可以看到在dist目录下的index.html是基于我们提供的模板生成：

![050-indexhtml-by-custom](https://static-res.zhen.wang/images/post/2022-12-31/050-indexhtml-by-custom.png)

此时，我们再次打开这个html，可以看到正确的处理后的结果：

![060-right-handle](https://static-res.zhen.wang/images/post/2022-12-31/060-right-handle.png)



# output.path与output.filename

让我们回到关于output.path与output.filename上来。回顾我们的webpack配置：

- output.filename：确定js最终生成的文件名

- output.path：确定js所在的根路径

js最终生成的路径是：

`output.path（绝对路径） + output.filename（文件名，可以有相对路径前缀）`

做一个简单的实验便可知，例如，我们修改配置如下，把output.filename改为`"js/main.js"`，output.path改为`'resolve(__diranme, "my-dist")'`：

```diff
     output: {
-        filename: 'main.js',
-        path: resolve(__dirname, 'dist')
+        filename: 'js/main.js',
+        path: resolve(__dirname, 'my-dist')
     },

```

重新经过构建以后，我们会看到`my-dist`目录被创建，并且这个目录下面还会创建`js`目录，`js`目录中会有main.js，正好匹配了`output.path（项目根目录/my-dist） + output.filename（js/main.js）`。

![070-new-output-path-and-filename](https://static-res.zhen.wang/images/post/2022-12-31/070-new-output-path-and-filename.png)

但是，output.filename与output.path仅仅影响js的生成吗？不然，让我看看这两个参数对于HtmlWebpackPlugin的关联关系。

## 与HtmlWebpackPlugin的关联

对于上述生成结果，我们会注意到，在webpack配置中的HtmlWebpackPlugin插件部分，我们没有编写过任何关于index.html的生成路径的配置，但这个index.html最终也生成到了`"my-dist"`目录下（与output.path一致）；此外，我们还可以发现，生成的index.html里面的script节点的src属性，是`"js/mian.js"`（与output.filename一致）。

我们可以整理一个图，来描述相关配置与js构建、HtmlWebpackPlugin插件的关联关系：![080-v1-path-filename](https://static-res.zhen.wang/images/post/2022-12-31/080-v1-path-filename.png)

总结来说，output.path与output.filename不能单纯只作为输出js的配置，HtmlWebpackPlugin也会使用它们：

- HtmlWebpackPlugin会使用**output.path** + **插件本身的filename配置**，作为html的生成路径；
- HtmlWebpackPlugin会使用**output.filename**作为生成的html中script节点src属性的js路径（**特别注意：这里还不准确，后续会补充修正！**）。

读者可以根据上述的表格，自己进行实验验证。

## 关于output.filename的注意点

对于output.filename，需要注意的是，不能是一个绝对路径，譬如：`"/js/main.js" or "/main.js"`，一旦配置成了绝对路径，就会看到报错：

```
configuration.output.filename: A relative path is expected. However, the provided value "/js/main.js" is an absolute path!
   Please use output.path to specify absolute path and output.filename for the file name.
```

你只能写成：`"js/main.js"`或`"./js/main.js"`。然而，由于生成的html中script节点属性src的值，来源于这个output.filename值，如果我们有需求，希望生成的src等于一个绝对路径，譬如：`src="/js/main.js"`，仅仅靠output.filename是不行的。于是乎，output.publicPath就登场了！

# output.publicPath

首先，在webpack中，这个参数不配置的话，默认是空字符串`""`。然后，我们需要**纠正我们前面的一个结论**：

- ~~HtmlWebpackPlugin会使用**output.filename**作为生成的html中script节点src属性的js路径~~

实际上，script节点的src属性的路径，并不只是output.filename来决定的，而是由output.publicPath与output.filename共同决定：

`src = output.publicPath（还有斜杠的特殊处理，后面讲）+ output.filename`

只是因为output.publicPath默认是空字符串，所以我们前面生成出来的只是`src="js/main.js"`。这里，我们可以做一个简单的实验，配置publicPath为`"/"`，则生成的节点就会成为：`<script src="/js/main.js">`

![090-publicPath-root](https://static-res.zhen.wang/images/post/2022-12-31/090-publicPath-root.png)

output.publicPath: "abc"（尾部没有"/"），src="abc/js/main.js"：

![100-publicPath-abc](https://static-res.zhen.wang/images/post/2022-12-31/100-publicPath-abc.png)

output.publicPath: "/abc"（尾部依然没有"/"），src="/abc/js/main.js"：

![110-publicPath-root-abc](https://static-res.zhen.wang/images/post/2022-12-31/110-publicPath-root-abc.png)

仔细观察这几种场景，就可以知道HtmlWebpackPlugin插件，在生成html中的script标签时候，其中的src属性依赖output.filename以及output.publicPath，并且规则为：

- publicPath为空白字符串（默认），则src="${output.filename}"；
- publicPath非空且不以"/"结尾，则src="${output.publicPath}/${output.filename}"（补充了一个"/"）；
- publicPath非空且以"/"结尾，则src="${output.publicPath}${output.filename}"；

需要注意的是，谨记js文件与html文件的生成不会受到output.publicPath的影响，只跟output.path和filename（js是output.filename，html是HtmlWebpackPlugin的filename）相关。

于是乎，我们重新整理前面的关系图，把output.publicPath配置引入：

![120-v2-path-filename-publicPath](https://static-res.zhen.wang/images/post/2022-12-31/120-v2-path-filename-publicPath.png)

细心的读者已经想到了，假如publicPath配置成了"/static/"，影响了HtmlWebpackPlugin中的script节点的src属性路径；而js文件实际生成路径仅受到output.path+output.filename，势必造成js访问路径不匹配的问题：

![130-path-not-match.png](https://static-res.zhen.wang/images/post/2022-12-31/130-path-not-match.png)

所以，日常对于webpack的配置一定要注意这种路径问题，保持匹配，否则使用webpack-dev-server就会出现问题～

相信看到这里，很多读者对output中的path、filename以及publicPath能够理解他们的效果了。接下来，我们举一反三，引入常用的CSS打包工具MiniCssExtractPlugin也来分析一下。

# 引入MiniCssExtractPlugin

我们通常会有这样的需求，一个前端项目打包的时候，希望能够将项目依赖的css文件最终抽离为一个或N个css文件，并让我们的前端html直接以link节点的形式加载。这个时候，我们一般使用MiniCssExtractPlugin来完成这个需求。当然，除了这个插件以外，我们还需要一个最基础的loader：`css-loader`。

```shell
yarn add -D css-loader mini-css-extract-plugin
```

工程结构不会变化：

```
项目根目录/
├─ package.json
├─ public
│    └─ index.html
├─ src
│    └─ index.js
└─ webpack.config.js
```

内容主要是新增了css-loader与mini-css-extract-plugin。

![140-project-content](https://static-res.zhen.wang/images/post/2022-12-31/140-project-content.png)

接下来，我们编写一个简单的css样式文件存放于src目录下（src/my-style.css）：

```css
body {
    background-color: aqua;
}

#app {
    background-color: azure;
}
```

并修改index.js的代码，在index.js中引用它：

```diff
+import './my-style.css';
 document.getElementById('app').innerText = 'hello, world'
```

此时，如果我们不进行任何的配置，运行webpack打包，会看到报错：

```
ERROR in ./src/my-style.css 1:5
Module parse failed: Unexpected token (1:5)
You may need an appropriate loader to handle this file type, currently no loaders are configured to process this file. See https://webpack.js.org/concepts#loaders
```

核心问题在于，webpack无法处理index.js中关于`.css`的文件（webpack默认值处理js文件）。所以，需要我们配置专门处理css的规则：

```diff
+ const MiniCssExtractPlugin = require('mini-css-extract-plugin');
module.exports = {
      ... ...
      plugins: [
         new HtmlWebpackPlugin({
             template: resolve(__dirname, 'public', 'index.html')
         }),
+        new MiniCssExtractPlugin({
+            filename: 'css/main.css'
+        })
     ],
+    module: {
+        rules: [{
+            test: /\.css/,
+            use: [MiniCssExtractPlugin.loader, 'css-loader']
+        }]
+    }
}
```

首先引入MiniCssExtractPlugin插件；然后在plugins中，new出MiniCssExtractPlugin插件实例，并传入filename配置`css/main.css`；最后，配置module.rules中，添加对css的处理：

>loader的执行顺序是按照数组从后向前的，所以use数组最后是css-loader，然后才是MiniCssExtractPlugin提供的loader。webpack在构建过程，遇到引用css的场景，则先调用css-loader，对css文件进行处理，然后调用MiniCssExtractPlugin提供的loader进行抽取

完成配置以后，我们再次启动webpack的构建，会看到dist目录下，又会产生一个css目录，里面存放的就是mian.js，并且，检查index.html会发现这一次除了script标签外，还插入了link标签：

![150-css-output](https://static-res.zhen.wang/images/post/2022-12-31/150-css-output.png)

有的读者可能已经能够推断出，这个link标签的href路径，也是根据output.publicPath+MiniCssExtractPlugin插件的filename组合而来。这里直接给出结论，就是这样的。我们再次更新图表，把导出css样式文件的MiniCssExtractPlugin插件与相关的配置关系也总结进去，得到如下最终版关系图：

![160-v3-path-filename-publicPath](https://static-res.zhen.wang/images/post/2022-12-31/160-v3-path-filename-publicPath.png)

# 关于关系图的补充

通过关系图，我们很容易知道，webpack中关于文件生成最核心的配置就是output.path以及各种filename，js的生成、css的生成、html的生成都依赖了这套配置；

其次，与js相关的output.filename和与css相关的MiniCssExtractPlugin.filename配置都有两个作用：

1. js、css的生成文件路径；
2. 被HtmlWebpackPlugin使用，以生成script节点和link节点中的资源路径（当然这个过程还有output.publicPath的参与）。

最后，本文并没有讲到webpack-dev-server和上述配置的关系，这个会在本《掌握webpack》系列中单独出一期。
