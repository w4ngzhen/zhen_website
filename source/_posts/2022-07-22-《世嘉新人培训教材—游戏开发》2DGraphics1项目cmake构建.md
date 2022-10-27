---
title: 《世嘉新人培训教材—游戏开发》2DGraphics1项目cmake构建
date: 2022-07-22
tags: 
 - game
 - sega
---

《世嘉新人培训教材—游戏开发》作为经典的游戏开发教程，提供了相关样例代码供我们进行开发使用。但是该样例是基于VS进行编写构建的，而本人日常喜欢CLion进行C/C++开发，于是准备使用cmake重新组织该书籍的样例项目：2DGraphics1中的NimotsuKunBox和drawPixels。当然，这个过程不仅是移植，也是对cmake组织项目一个深入的实践。

<!-- more -->

# 对现有样例项目的认识与构建

## 样例代码结构

在进行cmake迁移前，有必要对现有的VS体系的代码结构进行了解。

### GameLib（样例根目录）

![010-root](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/010-root.png)

该目录下主要存放了：

1. **各个样例会使用的工具静态库/头文件**；

2. **src**：样例源码；

3. **tools**：工具二进制程序。

### GameLib/src目录

![020-root-src](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/020-root-src.png)

该目录下主要存放：

1. **各种数字+下划线开头的文件夹**：书中使用到的各种样例工程；

2. **GameLibs文件夹**：生成GameLib根目录中的静态库/头文件的源码。

### GameLib/src/GameLibs目录

![030-root-src-GameLibs](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/030-root-src-GameLibs.png)

该目录主要存放：

1. **GameLib根目录下各个被样例项目使用的静态库/头文件的源码**；
2. **Modules**：其他静态库项目的依赖静态库。

## 使用VS构建样例项目静态库

在GameLib下，本书的译者已经帮我们编写了一个基本的指南：

>编译顺序
>在系统环境变量中添加 GAME_LIB_DIR 值为源码工程的根目录
>注意要重启visual studio
>
>①先编译类库的Modules
>src\GameLibs\Modules\Modules.sln
>
>②再编译各个小功能的类库
>比如 src\GameLibs\2DActionGame\GameLib.sln
>
>③最后编译游戏本身
>比如 src\01_FirstGame\FirstGame.sln
>
>为什么要按照这样的顺序呢？请看下面这个例子
>譬如对src\02_2DGraphics1\2DGraphics1.sln 来说，
>首先用vs打开它，右键点击 drawPixels查看属性
>在链接器 的附加库目录一栏可以看到  $(GAME_LIB_DIR)\2DGraphics1\lib;%(AdditionalLibraryDirectories)
>这意味着它需要在2DGraphics1\lib中查找某些类库，
>具体要用什么类库呢？可以点击 链接器 -> 输入 ，看到附加依赖项中有 GameLib_d.lib;%(AdditionalDependencies)
>
>如何才能生成这个 GameLib_d.lib呢?
>打开 src\GameLibs\2DGraphics1\GameLib.sln 编译即可
>但是，通过右键Framework属性， 查看库管理器 的附加依赖项可以看到 Modules_d.lib
>这就要求必须先编译好 Modules工程
>于是打开 src\GameLibs\Modules\Modules.sln 编译即可。

这里有两个关键点需要牢记：

1. 需要配置环境变量`GAME_LIB_DIR`，原因在于后续即将编译的各个样例，都会使用`$(GAME_LIB_DIR)`然后找到对应的类库；
2. 编译有一个顺序：先核心静态库：Modules；然后各个样例需要的GameLib；最后是样例。

### 编译核心的Modules

加载`$(GAME_LIB_DIR)\src\GameLibs\Modules\Modules.sln`进行构建

使用vs编译后会生成`$(GAME_LIB_DIR)\src\GameLibs\Modules\lib\Modules_d.lib`（`_d`代表Debug的静态库）

### 编译各独立样例需要的GameLib

在本文中，我们的目标是构建2DGrphics1-NimotsuKunBox项目，所以我们加载2DGrphics1的GameLib：`$(GAME_LIB_DIR)\src\GameLibs\2DGraphics1\GameLib.sln`进行构建，通过配置也能看到的确需要`Modules_d.lib`：

![040-gamelib-use-modules](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/040-gamelib-use-modules.png)

该项目构建完成后，会生成：`$(GAME_LIB_DIR)\2DGraphics1\lib\GameLib_d.lib`，并且会将`GameLib_d.lib`静态库以及相关头文件都复制到`$(GAME_LIB_DIR)\2DGraphics1\中`，：

```
$(GAME_LIB_DIR)\2DGraphics1\
├─include
│  └─GameLib
│      │  Framework.h
│      │
│      └─Base
│          DebugStream.h
│
└─lib
        Framework_d.idb
        Framework_d.pdb
        GameLib_d.lib（lib库）
        GameLib_d.pdb
        Modules_d.pdb
```

目前为止，我们生成了如下的两静态库以及头文件：

1. `$(GAME_LIB_DIR)\src\GameLibs\Modules\lib\Modules_d.lib`
2. `$(GAME_LIB_DIR)\2DGraphics1\lib\GameLib_d.lib`
3. `$(GAME_LIB_DIR)\2DGraphics1\include（头文件）`

**当然，因为我们的`GameLib_d.lib`是使用`Modules_d.lib`进行构建的，已经将`Modules_d.lib`链接到了`GameLib_d.lib`内部了，所以接下来我们的cmake项目不再需要`Modules_d.lib`了。**

### 演示2DGraphics1-NimotsuKunBox和drawPixels项目

使用VS打开`$(GAME_LIB_DIR)\src\02_2DGraphics1\2DGraphics1.sln"`解决方案，该解决方案中有如下的5个样例：

```
NimotsuKunBox
NimotsuKunBoxWithTermination
NimotsuKunTextOnly
NimotuKunDot
drawPixels
```

将NimotsuKunBox项目作为启动项目，然后运行可以看到如下的界面：

![050-NimotsuKunBox-show](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/050-NimotsuKunBox-show.png)

将drawPixels作为启动项，运行可看到如下效果：

![060-drawPixels-show](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/060-drawPixels-show.png)

接下来，我们将使用cmake来移植这两个项目。

# 使用cmake搭建2DGraphics1项目

在经过前戏后，我们终于编译出了2DGraphics1所需要的`GameLib_d.lib`静态库以及相关的头文件，并且，我们还构建了2DGraphics1样例解决方案中的NimotsuKunBox和drawPixels项目。接下来我们将创建一个cmake项目，移植该样例中的两个项目。

## 搭建初始项目

首先，我们建立一个文件夹2DGraphics1_cmake，在该文件夹中，我们再创建两个文件夹：NimotsuKunBox和：drawPixels，并且这两个文件夹中分别各自创建一个main.cpp文件和CMakeLists.txt，内容如下：

2DGraphics1_cmake/NimotsuKunBox：

```c++
// 2DGraphics1_cmake/NimotsuKunBox/main.cpp
#include <iostream>
int main() {
    std::cout << "hello, NimotsuKunBox" << std::endl;
}
```

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.22)
PROJECT(NimotsuKunBox)

SET(CMAKE_CXX_STANDARD 11)

ADD_EXECUTABLE(NimotsuKunBox main.cpp)
```

2DGraphics1_cmake/drawPixels：

```cpp
// 2DGraphics1_cmake/drawPixels/main.cpp
#include <iostream>
int main() {
    std::cout << "hello, drawPixels" << std::endl;
}
```

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.22)
PROJECT(drawPixels)

SET(CMAKE_CXX_STANDARD 11)

ADD_EXECUTABLE(drawPixels main.cpp)
```

接着，我们在2DGraphics1_cmake中创建一个CMakeLists.txt来统一管理NimotsuKunBox和drawPixels这两个cmake项目，内容如下：

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.22)
PROJECT(2DGraphics1_cmake)

SET(CMAKE_CXX_STANDARD 11)

# cmake子项目，子项目的名称就是子目录下的CMakeLists.txt中的PROJECT一致的名称
ADD_SUBDIRECTORY(NimotsuKunBox)
ADD_SUBDIRECTORY(drawPixels)
```

于是，当前整体目录结构如下：

```
2DGraphics1_cmake
├─ CMakeLists.txt
├─ NimotsuKunBox
│    ├─ CMakeLists.txt
│    └─ main.cpp
└─ drawPixels
     ├─ CMakeLists.txt
     └─ main.cpp
```

> cmake实践点：子项目管理
>
> 父级CMakeLists.txt可以通过ADD_SUBDIRECTORY来添加子CMake项目。这里有一篇特别详细的博文[CMake基础 第13节 构建子项目 - 橘崽崽啊 - 博客园 (cnblogs.com)](https://www.cnblogs.com/juzaizai/p/15069693.html)

## 头文件与静态库添加

在前面我们已经编译出了`GameLib_d.lib`，并且把头文件已经复制到了指定目录。现在，我们在项目根目录中创建一个`@lib`文件夹，专门放置静态库和头文件，目录状态如下：

```
2DGraphics1_cmake
├─ CMakeLists.txt
├─ NimotsuKunBox
│    └─ // ... ...
├─ drawPixels
│    └─ // ... ...
└─ @lib  // => 存放要使用的静态库和头文件
    ├─ include
    │    └─ GameLib
    │           ├─ Base
    │           │    └─ DebugStream.h
    │           └─ Framework.h
    └─ lib
           ├─ Framework_d.idb
           ├─ Framework_d.pdb
           ├─ GameLib_d.lib
           ├─ GameLib_d.pdb
           └─ Modules_d.pdb
```

## 配置NimotsuKunBox项目

为了让NimotsuKunBox项目中的能够使用到根目录下的静态库和头文件，我们需要配置NimotsuKunBox/CMakeLists.txt，添加头文件和静态库：

```diff
  SET(CMAKE_CXX_STANDARD 11)
 
+ MESSAGE("Current CMakeLists.txt dir:\n${CMAKE_CURRENT_SOURCE_DIR}")
+
+ # 配置头文件查找路径
+ INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/../@lib/include)
+ # 配置链接库文件查找路径
+ LINK_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/../@lib/lib)
+
  ADD_EXECUTABLE(NimotsuKunBox main.cpp)
+
+ # 实际链接
+ TARGET_LINK_LIBRARIES(NimotsuKunBox GameLib_d.lib)
```

之后，我们将在VS中能够运行的NimotsuKunBox项目代码拷贝到当前的main.cpp中，由于篇幅的关系，就不贴出代码本身了，给一个整体的修改：

![070-NimotsuKunBox-modified01](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/070-NimotsuKunBox-modified01.png)

### 编译问题

当我们尝试运行该项目的时候，发现至少有以下几个问题：

**问题1：在CLion+msvc编译器下，编码字符报错：warning C4819: 该文件包含不能在当前代码页(936)中表示的字符。请将该文件保存为 Unicode 格式以防止数据丢失。**

该问题原因在于CLion中的文件是默认使用的UTF-8编码，而msvc在不指定的情况默认以当前代码页（936）编码方式读取文件（*代码页936（Codepage 936）是Microsoft的简体中文字符集标准，是东亚语文的四种双字节字符集（DBCS）之一。其最初版本和GB 2312一模一样，但在推出Windows 95时扩展成GBK*）。

在CMake中想要给msvc指定文件编码方式，需要在CMakeLists.txt配置如下内容：

```cmake
... ...
SET(CMAKE_CXX_STANDARD 11)

# 配置编译器以指定编码读取代码源文件
ADD_COMPILE_OPTIONS("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
ADD_COMPILE_OPTIONS("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

MESSAGE("Current CMakeLists.txt dir:\n${CMAKE_CURRENT_SOURCE_DIR}")
... ...
```

**问题2：GameLib_d.lib(MemoryManager.obj) : error LNK2038: 检测到“RuntimeLibrary”的不匹配项: 值“MTd_StaticDebug”不匹配值“MDd_DynamicDebug”(main.cpp.obj 中)**

这一类报错通常比较普遍，简单来讲就是：GameLib_d.lib这个库是一个静态库带Debug（MTd_StaticDebug），但是我们的项目链接步骤是以动态库的方式链接这些库文件。对于这个问题，有两种方式来解决，一种就是重新编译GameLib为一个dll（动态链接库）；另一种则是修改当前项目的链接方式为静态库链接。当然，简便起见，我们修改项目的链接形式为静态库链接形式：

```diff
 ... ...
 SET(CMAKE_CXX_STANDARD 11)
 ADD_COMPILE_OPTIONS("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
 ADD_COMPILE_OPTIONS("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
 
+# 设置策略CMP0091为NEW，新策略
+if (POLICY CMP0091)
+    CMAKE_POLICY(SET CMP0091 NEW)
+endif (POLICY CMP0091)
+
 MESSAGE("Current CMakeLists.txt dir:\n${CMAKE_CURRENT_SOURCE_DIR}")
 
 LINK_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/../@lib/lib)
 
 ADD_EXECUTABLE(NimotsuKunBox main.cpp)
 
+# 设置MT/MTd
+SET_PROPERTY(
+        TARGET NimotsuKunBox
+        PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
+
 # 实际链接
 TARGET_LINK_LIBRARIES(NimotsuKunBox GameLib_d.lib)
```

关于这块配置的细节，可以参考这篇文章：[CMake设置MSVC工程MT/MTd/MD/MDd_Copperxcx的博客-CSDN博客_cmake mt](https://blog.csdn.net/Copperxcx/article/details/123084367)

**问题3：error LNK2019: 无法解析的外部符号 _main，函数 "int __cdecl invoke_main(void)" (?invoke_main@@YAHXZ) 中引用了该符号**

稍有C/C++开发经验的开发者看到这个报错其实心里还是有底的，应该是没有提供main函数作为函数的入口。但是对于我们的项目，细心的读者发现似乎样例代码中确实是没有提供main入口函数的。那么，为什么vs项目能够正确运行起来呢？观察vs中的项目属性—连接器—系统，会发现子系统（SubSystem）的值是：`/SUBSYSTEM:WINDOWS`

![070-subsystem-windows](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-07-22-sega-game-dev/080-subsystem-windows.png)

在cmake项目中，我们可以按照如下的方式进行配置：

```diff
# 设置MT/MTd
SET_PROPERTY(
        TARGET NimotsuKunBox
        PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

+ # 配置 "/SUBSYSTEM:WINDOWS"
+ TARGET_LINK_OPTIONS(NimotsuKunBox PRIVATE "/SUBSYSTEM:WINDOWS")

# 实际链接
TARGET_LINK_LIBRARIES(NimotsuKunBox GameLib_d.lib)
```

实际上，配置成了`/SUBSYSTEM:WINDOWS`之后也是需要有一个入口函数的，**这个入口函数其实是在Modules那个项目里面定义好了的**，具体可以搜索Modules项目中的`int APIENTRY _tWinMain`函数实现：

```c++
int APIENTRY _tWinMain(HINSTANCE hInstance,
                     HINSTANCE hPrevInstance,
                     LPTSTR    lpCmdLine,
                     int       nCmdShow)
```

上述问题处理完成后，我们通过项目编译

## 配置drawPixels项目

实际上，配置drawPixels项目的CMakeLists.txt和NimotsuKunBox的CMake配置结构上没有区别，只是需要把相关的项目名字等换位drawPixels即可。最终运行的效果和之前的vs下是一致的~

# 附录：项目地址

本cmake移植的项目地址在：[w4ngzhen/2DGraphics1_cmake (github.com)](https://github.com/w4ngzhen/2DGraphics1_cmake)