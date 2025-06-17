---
title: 从源码分析node-gyp指定node库文件下载地址
date: 2021-05-12
tags:
 - node-gyp
categories: 
- 技术
---

当我们安装node的C/C++原生模块时，涉及到使用node-gyp对C/C++原生模块的编译工作（configure、build）。这个过程，需要nodejs的**头文件**以及**静态库**参与（后续称库文件）对C/C++项目编译和链接。库文件从哪里下载，会有一定逻辑进行处理，本文将从源码入手进行分析。

<!-- more -->

# 编写简单的原生模块

为了方便进行分析，我们首先创建一个原生模块（关于如何编写原生模块的细节不再本文讨论）。

**hello_world.cc**

```c
#include <node.h>

void Method(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(v8::String::NewFromUtf8(
      isolate, "world").ToLocalChecked());
}

void Initialize(v8::Local<v8::Object> exports) {
  NODE_SET_METHOD(exports, "hello", Method);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)
```

**binding.gyp**

```
{
  "targets": [
    {
      "target_name": "hello_world",
      "sources": [ "hello_world.cc" ]
    }
  ]
}
```

**index.js**

```js
const binding = require('./build/Release/hello_world');

console.log(binding.hello());
```

**package.json**

```json
...  
  "scripts": {
    "build": "node-gyp configure && node-gyp build",
    "run:demo": "node index.js"
  },
...
```

**整体结构**

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/node-addon-simple-demo-proj-arch.jpg)

按照如下命令依次运行：

```shell
$ npm run build
// 使用node-gyp配置并构建
$ npm run run:demo
// 运行Demo
```

输出如下：

```bash
D:\Projects\node-addon-demo>npm run run:demo

> node-addon-demo@1.0.0 run:demo
> node index.js

world
```

# 从源码分析node-gyp下载库文件的路径

首先要直接给出一个结论，库文件并不是每次都要从网络上下载，库文件下载后会缓存在本地一个目录，在Windows上为`C:\Users\用户\AppData\Local\node-gyp\Cache`中，并按照nodejs的版本进行存储：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/node-headers-cache-dir.jpg)

*本人电脑安装的node版本为14.15.0，且曾经已经缓存了对应的库文件。*

为了便于分析，我们首先删除该缓存文件，并且在原有的npm命令加上`--verbose`，输出更加详细的日志：

```
$ npm run build --verbose
```

于是，我们可以从众多的输出中，看到一个关键信息：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/show-official-dist-url.jpg)

从日志中可以看出，node-gyp在构建过程中，会创建缓存目录，然后从指定URL下载指定版本的headers文件。

我们利用GrepWin（一款Windows下超好用的文本内容搜索工具，[官网](https://tools.stefankueng.com/grepWin.html)），在node-gyp目录中搜索`created nodedir`这个关键词，因为可以看到`gyp http GET`上面出现了这个关键词。那么现在有一个新的问题，node-gyp目录在哪儿？其实，从上面的日志往上查看，能够找到：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/where-is-node-gyp.jpg)

这里是调用的我们全局安装的npm依赖的node-gyp，于是我们定位到node-gyp所在目录进行搜索：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/search-create-nodedir.jpg)

进入该文件，我们找到：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/create-nodedir-and-download.jpg)

找到关键词搜索后，继续往后续代码查阅，能够看到一个`download`函数的调用，入参最后一位是url，此时已经是成型的url，所以接下来我们需要确定，`release.tarballUrl`这个值，究竟是什么时候确定的。

## tarballUrl如何得到

继续向上翻阅代码，能够在入口处看到这个release是如何生成的：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/get-the-release.jpg)

进入代码后，能够找到一段核心的构建：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/distUrl-make-flow.jpg)

通过上述代码流程，我们总结出来，tarballUrl的baseUrl取决于是否存在overrideDistUrl，若存在，则直接使用；否则使用默认URL：`https://nodejs.org/dist`。

再查看`overrideDistUrl`的传入点：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/overrideDistUrl.jpg)

也就是说，gyp对象的opts属性存在`dist-url`或`disturl`时，就会使用该值作为库文件下载的baseUrl。

## 如何构建gyp.opts

首先检查该函数的调用点：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/where-use-processRelease.jpg)

发现configure和install.js都使用了该函数，且都是入口处进行的调用的：

**configure.js**

```js
function configure (gyp, argv, callback) {
  var python
  var buildDir = path.resolve('build')
  var configNames = ['config.gypi', 'common.gypi']
  var configs = []
  var nodeDir
  var release = processRelease(argv, gyp, process.version, process.release)
  ......
}
module.exports = configure
```

**install.js**

```js
function install (fs, gyp, argv, callback) {
  var release = processRelease(argv, gyp, process.version, process.release)
  ......
}
module.exports = function (gyp, argv, callback) {
  return install(fs, gyp, argv, callback)
}
```

可以看到confiigure.js和install.js都作为函数形式导出，也就是说，gyp这个对象是在这两个模块在被导入并以函数形式调用时被传入的。那么接下来我们需要看这两个模块在何处使用的。

在上文我们查看当前执行的node-gyp目录的时候，我们就看到过：

```
gyp verb cli [
gyp verb cli   'D:\\Programs\\nodejs\\node.exe',
gyp verb cli   'D:\\Programs\\nodejs\\global_modules\\node_modules\\npm\\node_modules\\node-gyp\\bin\\node-gyp.js',
gyp verb cli   'configure'
gyp verb cli ]
```

入口函数是：`node-gyp根目录/bin/node-gyp.js`。所以，我们将node-gyp以项目的形式添加到IDEA中，尝试以相同的形式调用这些命令，通过开启DEBUG模式，来一探究竟。

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/debug-node-gyp-portal.jpg)

在`bin/node-gyp.js`中的最下方进行了一个名为`run`的函数调用：

```js
// bin/node-gyp.js

// ......
// 还有很多省略的代码......

// start running the given commands!
run()
```

根据注释可以，`run()`执行所提供的命令。翻阅该函数：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/the-run-func.jpg)

总体分为两步：

1. 从对象prog的todo这个数组中取出首个command命令对象，不存在判定为所有命令执行完成。
2. 从对象prog的命令数组（commands）中找到对应命令名称（command.name），通过代码可知，该命令实际上对应一个函数。传入参数（command.args）完成该函数的调用。

那么这个prog是什么呢？通过向上阅读代码，可以知道来自于上层目录提供的模块：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/what-is-gyp.jpg)

而上层所指代的模块是通过package.json的`main`字段可知是`lib/node-gyp.js`：

```json
// 根目录下的package.json
"main": "./lib/node-gyp.js",
```

进入该文件的gyp函数，返回的是类Gyp的实例，而Gyp实例的构造过程如下：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/Gyp-constructor-and-commands.jpg)

1. 使用self变量指代Gyp实例，并创建devDir和commands字段。
2. 遍历上方的commands字符串数组，给self（也就是Gyp实例）的commands属性中，逐步添加对应命令名称的函数，函数的实现是：require和command同名的js模块，这些模块又本身是以函数形式导出的，最终是调用对应模块函数。举例说明：当遍历到command为`configure`的时候，就是如下的形式：

```js
  self.commands['configure'] = function (argv, callback) {
    log.verbose('command', 'configure', argv)
    return require('./configure')(self, argv, callback)
  }
```

那么在进行`node-gyp configure`时的调用栈就如下：

```js
执行node-gyp configure:
=> run()
...
...
=> gyp.commands['configure'](argv, cb);
=> require('./configure')(self, argv, cb); // self就是Gyp实例
```

前文我们已经知道了configure.js这个模块导出的就是一个函数：

```js
// configure.js
function configure (gyp, argv, callback) {
  var python
  var buildDir = path.resolve('build')
  var configNames = ['config.gypi', 'common.gypi']
  var configs = []
  var nodeDir
  // 这个gyp，就是入参gyp，也就是上面的gyp实例
  var release = processRelease(argv, gyp, process.version, process.release)
  ... ...
}
... ...
module.exports = configure
```

所以，我们终于知道`processRelease`的入参的gyp，就是上面的gyp实例。那么gyp实例中的opts属性，是哪儿来的呢？使用IDEA的Debug进行断点调式，调试`bin/node-gyp.js`：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/before-handle-opts.jpg)

可以看到，在执行`parseArgv`这个函数前，gyp实例里面还不存在opts属性，而执行后，又在使用opts属性的devdir。也就是说，`parseArgv`这个函数一定构建了opts，接下来我们重点分析这个函数。

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/parseArgv-portal.jpg)

入口的argv就是我们的运行时入参：

```
"dev": "node ./bin/node-gyp.js configure"
```

首先会经过`nopt`函数，看样子，是对命令行参数以及短命令的处理：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/opts-build-1.jpg)

然后是该函数其他的部分：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/opts-build-2-handle-argv-and-env.jpg)

主要分为两个部分：

1. 对argv的解析
2. 对环境变量的解析

对argv的解析不涉及设置opts属性，我们重点看对环境变量的解析：

```js
  // support for inheriting config env variables from npm
  var npmConfigPrefix = 'npm_config_'
  Object.keys(process.env).forEach(function (name) {
    if (name.indexOf(npmConfigPrefix) !== 0) {
      return
    }
    var val = process.env[name]
    if (name === npmConfigPrefix + 'loglevel') {
      log.level = val
    } else {
      // add the user-defined options to the config
      name = name.substring(npmConfigPrefix.length)
      // gyp@741b7f1 enters an infinite loop when it encounters
      // zero-length options so ensure those don't get through.
      if (name) {
        this.opts[name] = val
      }
    }
  }, this)
```

处理流程为：

1. 判断环境变量的名称（name），如果**不是**以`npm_config_`开头，则跳过该次处理，否则进入下一步。
2. 如果变量名是`npm_config_loglevel`（npm的日志等级变量），则使用该日志等级作为node-gyp在使用npm时候的日志变量（这是对日志等级的特殊处理）。
3. 否则（一般处理），截断该变量的名，例如`name = 'npm_config_my_key'`，则得到`my_key`，设置到opts中：`opts['my_key'] = 变量值`。

至此，我们已经知道了，opts属性的值来源于上述的解析。

那么，回到我们一开始的目的，我们知道了要实现从指定的地方下载node的库文件，只要opts里面存在`dist-url`或是`disturl`即可。有些读者可能会说，那这样就行了呀：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/custom-url-by-set-dist-url.jpg)

实际上，并不行：

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/just-has-dist_url.jpg)

解析结束后，会发现，gyp.opts中是不存在`dist-url`字段的，只有`dist_url`。这一切的缘由，都是因为，npm在处理环境变量的时候，会将`-`替换为下划线`_`（[config | npm Docs (npmjs.com)](https://docs.npmjs.com/cli/v7/using-npm/config#environment-variables)）。

好在，node-gyp还能够处理opts中的`disturl`字段。所以我们只需要在使用npm来使用node-gyp的时候，加入参数`--disturl`。现在，让我们回到我们一开始的node-addon-demo，添加设置变量的参数：

```json
  "scripts": {
    "build": "node-gyp configure && node-gyp build",
    "build:custom": "npm run build --verbose --disturl=this_is_my_custom_url",
    "run:demo": "node index.js"
  },
```

上述`build:custom`就是我们新加的配置，通过运行，果然，加载的是我们制定的url：

```
gyp verb created nodedir C:\Users\w4ngzhen\AppData\Local\node-gyp\Cache\14.15.0
// 这里报错忽略，因为使用的是一个无效的url: 'this_is_my_custom_url'
// 主要是为了验证确实是改变了
gyp http GET this_is_my_custom_url/v14.15.0/node-v14.15.0-headers.tar.gz
gyp WARN install got an error, rolling back install
gyp verb command remove [ '14.15.0' ]
```

## node-gyp的直接使用和npm使用的区别

那么，有的细心的读者可能会说，明明这里通过npm使用的时候会转为下划线，那在node-gyp的官方github，说是可以使用`dist-url`这个参数呢？。

[nodejs/node-gyp: Node.js native addon build tool (github.com)](https://github.com/nodejs/node-gyp)

![](https://res.zhen.wang/images/post/2021-05-12-node-gyp-dist-url/official-dist-url.jpg)

实际上，官方文档给出的参数，需要你直接使用node-gyp方式进行设置，也就是说，--dist-url这个参数必须紧跟node-gyp的命令：

```
node-gyp configure --dist-url=xxx
```

像是上面的`npm run ${使用node-gyp的脚本名} --dist-url=xxx`，这个dist-url是作为npm的参数来被识别，而非node-gyp。所以，对于demo，我们还可以如下：

```json
  "scripts": {
    "build": "node-gyp configure --dist-url=this_is_my_custom_url && node-gyp build --dist-url=this_is_my_custom_url",
    "build:custom": "npm run build --verbose",
    "run:demo": "node index.js"
  },
```

注意，这一次，我把`--dist-url`是放在和node-gyp命令的参数的。但是，我们知道有些npm包，内部就直接使用node-gyp进行配置编译的操作，这个过程没法通过`--dist-url`紧跟`node-gyp`命令方式，所以只能在例如`.npmrc`文件中配置兼容的不会被下划线处理的`disturl`。

# 总结

要想让node-gyp下载node库文件的时候，能够走指定的镜像，可以通过配置`--dist-url`或是`--disturl`的方式，但配置`dist-url`形式参数只能是参数紧跟`node-gyp`的形式：

```
node-gyp configure --dist-url=xxx
```

而**不能**是如下的形式：

```
// 你的package.json scripts字段
"build": "node-gyp configure"
// 然后在命令行调用
npm run build --dist-url=xxx // 
```

因为此时`--dist-url`参数是npm的参数，且会被处理为`npm_config_dist_url`下划线形式，进而在gyp.opts只有dist_url属性。

所以，最安全的方式是使用disturl参数：

情况1：

```
node-gyp configure --disturl=xxx
```

情况2：

```
// 你的package.json scripts字段
"build": "node-gyp configure"
// 然后在命令行调用
npm run build --disturl=xxx
```

情况1下，disturl是作为node-gyp的参数进行解析，能够被设置到opts中。

情况2，disturl是作为npm的参数被加入到npm环境变量：`npm_config_disturl`，此时，node-gyp解析process.env的时候，也能解析到`disturl`进而设置到opts。