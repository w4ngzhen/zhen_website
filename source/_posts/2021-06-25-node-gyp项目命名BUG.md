---

title: node-gyp项目命名BUG
date: 2021-06-25
tags:
 - node-gyp
 - C/C++

---

当我们编写node原生模块的时候，免不了对node-gyp项目进行命名，在node-gyp进行build的时候，会跟binding.gyp配置文件中的target_name生成对应的原生模块。但是，如果target_name填写不规范，会触发编译问题。

<!-- more -->

# 问题与解决

本人发现，当target_name使用了短中线的时候（"-"），会导致编译过程中触发编译问题：

```
 error C2143: 语法错误: 缺少“;”(在“-”的前面) 
```

使用下划线命名以及各种驼峰命名不会出现此问题。出现问题的点为文件最后使用宏的时候：

```
NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)
```

解决方案，target_name名称不使用中横线：

```
target_name: "the-demo" => target_name: "theDemo"
或
target_name: "the-demo" => target_name: "the_demo"
```

# 问题分析

接下来的问题分析，需要一定的C/C++知识。

## 编写样例

这里不再赘述样例，直接使用这篇文章建立一个demo：[使用node-gyp编写简单的node原生模块 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/383948462)，或是[使用node-gyp编写简单的node原生模块 - w4ngzhen - 博客园 (cnblogs.com)](https://www.cnblogs.com/w4ngzhen/p/14931367.html)。

Demo编写完成后，我们修改其中的target_name，使其带有中横线（"-"）：

```
{
  "targets": [
    {
      "target_name": "hello-world",
      "sources": [ "hello_world.cc" ]
    }
  ]
}
```

修改为该target_name后，我们进行`node-gyp configure && node-gyp build`，会发现编译器报错：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/compile-error.jpg)

## 使用IDE分析

我们曾经讲过，node-gyp实际上只是构建工具，他会根据各个操作平台，生成对应平台的项目。在Windows上，它最终会帮你生成一个解决方案。查看项目目录下，我们就能看到一个build文件夹，这个文件夹下面会有解决方案：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/node-gyp-build-sln.jpg)

我们使用VS打开，开始进行分析：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/demo-sln-in-vs.jpg)

通过IDE的智能提示，我们看到在下面的宏使用报错了：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/compile-err-in-vs.jpg)

通常，对于宏报错，我们需要的第一步是进行宏展开，查看到底是什么导致了编译错误的。在VS中，我们进行进行如下的配置，让编译器首先生成宏展开的源码：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/gen-i-file.jpg)

然后，我们重新进行编译，可以看到在对应的生成目录下，产生了一个`.i`后缀的文件。

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/i-file-generated.jpg)

这个宏展开后的源码文件，可以更见方便的便于我们分析。我们直接定位到这个文件的最下方，可以看到我们已经经过宏展开的代码：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/i-file-content.jpg)

我们67404这行宏展开的代码拷贝到VS对应宏使用的地方，通过IDE来更加智能的检查这段有何问题：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/replace-macro-to-code.jpg)

因为改行很长，这里我进行一下格式化代码的操作：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/macro-unfold-error-point.jpg)

可以看到，宏展开里面模块名为"hello-world"，在上图指出的部分，被分割为了"hello - world"，而分割开来后，导致了语法错误。如果target_name使用的"hello_world"，则不会有这个问题：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-06-25-node-gyp-target-name-bug/macro-unfold-with-underline.jpg)

实际上被`"-"`分割，是因为在宏展开的时候，作为了函数名的一部分，而函数名标识符是不能有`"-"`的。这里举例：

```c
#define NAME hello-world

#define TEST_MACRO(fn) static void fn(void);

TEST_MACRO(NAME) // 报错，因为最终展开后：static void hello-world(void);

int main()
{
    return 0;
}
```

>C语言规定，标识符只能由字母（A~Z, a~z）、数字（0~9）和下划线（_）组成，并且第一个字符必须是字母或下划线，不能是数字。

所以这就是为什么target_name使用有中横线的名称会报错了。
