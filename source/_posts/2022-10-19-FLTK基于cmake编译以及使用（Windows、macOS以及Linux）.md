---
title: FLTK基于cmake编译以及使用（Windows、macOS以及Linux）
date: 2022-10-19
tags:
 - fltk
 - gui
---

最近因为一些学习的原因，需要使用一款跨平台的轻量级的GUI+图像绘制 C/C++库。经过一番调研以后，最终从GTK+、FLTK中选出了FLTK，跨平台、够轻量。本文将在Windows、macOS以及Linux Debian三套操作系统环境，对FLTK进行编译，并搭建简单Demo。这其中也有少许的坑，也在此文进行记录。

<!-- more -->

# 前期准备

- FLTK 1.3.8（最新稳定版）[FLTK 1.3.8: FLTK Programming Manual](https://www.fltk.org/doc-1.3/index.html)
- CMake 3.5+
- Windows 11（VS2022）/ macOS 12.6 / Linux Debian 11
- CLion工具

PS：后续操作系统差异带来的配置/代码差异我会特别指明

# 编译FLTK

## 编译静态库文件

首先从官方地址下载FLTK 1.3.8代码：[Download - Fast Light Toolkit (FLTK)](https://www.fltk.org/software.php?VERSION=1.3.8)。

下载完成后，将文件内容解压至某个自定义目录，例如在macOS下，我存放在`"用户主目录/Projects/third-lib-projects/fltk-1.3.8"`目录。

进入该目录后，我们创建一个build目录，并进入build目录，然后使用CMake进行**配置**。

上述一些列命令操作整理如下：

```shell
# 进入fltk跟目录
cd "用户主目录/Projects/third-lib-projects/fltk-1.3.8"
# 创建 build 文件夹
mkdir build
# 进入 build 文件夹
cd build
# 执行 cmake 配置操作（注意，cmake后面跟空格，再跟".."，cmake中"外部构建"方式）
# 执行该命令前，请先阅读下面的cmake前置条件
cmake ..

# Windows下建议使用PowerShell，上述的命令基本没有差别。
```

### cmake配置前置条件

**Windows**

无

**macOS**

无

**Linux**

在Linux下，使用cmake进行项目生成前，务必确保一些基础库的安装：

```shell
# 安装gcc/g++等核心开发构建工具和库（必备）
sudo apt-get install build-essential
# openGL库安装（可选，建议。OpenGL的Library、Utilities以及ToolKit）
sudo apt-get install libgl1-mesa-dev
sudo apt-get install libglu1-mesa-dev
sudo apt-get install freeglut3-dev
# openSSL库（可选）
sudo apt-get install libssl-dev
# x11库（必备）
sudo apt-get install libx11-dev
```

对于必备的X11库，安装完成后是以动态链接库（`.so`）的方式存放于`/usr/lib/x86_64-linux-gnu/libX11.so`。在上面的`cmake ..`命令执行后，你也会看到控制台输出的一些关键内容：

```
# cmake .. 执行后的输出内容
... ...
-- Looking for XOpenDisplay in /usr/lib/x86_64-linux-gnu/libX11.so; ... - found
... ...
```

### 调用对应平台工具链完成FLTK编译

cmake进行项目构建完成后，在我们当前的build目录中，对于macOS/Linux类操作系统，CMake会为我们生成了对应的makefile文件，所以我们直接使用make命令调用本地的clang或则gcc进行编译。

```shell
# 在build目录下，默认就是release版
make
```

在Windows操作系统，请直接使用vs打开build中的解决方案FLTK.sln，打开后对项目`ALL_BUILD`进行Release模式编译。

编译完成后，build目录中会生成一个lib文件夹，这里面存放的就是fltk编译出来的静态链接库。

在macOS/Linux中是`lib`前缀，`.a`结尾：

```
# macOS/Linux（Release模式）
build/lib
├── libfltk.a
└── ... ...
```

在windows中是`.lib`结尾：

```
# Windows（Release模式文件名结尾不会有"d"）
build/lib
└── Release
    ├── fltk.lib
    └── ... ...
```

## 准备头文件

对于我的方式，在build文件夹中，我们创建一个inlude文件夹，并且将build上一层的fltk根目录中的FL文件夹复制到build/include中，形成如下结构：

```
build/include
└── FL
    ├── Enumerations.H
    ├── FL.H
    └── ... ...
```

**注意**，请同时务必将build目录下`FL/abi-version.h`复制到`include/FL/abi-version.h`。原因在于`FL.h -> Enumerations.h`头文件会用到该头文件里面的一些定义，不添加则会报错：

```
fatal error: 'FL/abi-version.h' file not found
#include <FL/abi-version.h>
         ^~~~~~~~~~~~~~~~~~
```

## 库、头文件准备总结

至此，我们准备好了：

1. build/lib下的fltk相关静态库（`.a/.lib` ）；

2. 在build/include里面的头文件。

为了方便我们后续的使用，创建一个文件夹`fltk-dist-1.3.8`，并将lib、include复制到dist中：

```
fltk-dist-1.3.8/
    include
    └── FL
        ├── Enumerations.H
        ├── FL.H
        └── ... ...
    lib
    └── macOS-release
        ├── libfltk.a
        └── ... ...
    └── linux-release
        ├── libfltk.a
        └── ... ...
    └── Windows-release
        ├── fltk.lib
        └── ... ... 
```

# 基础项目搭建

创建一个名为fltk-demo目录

1. 将上一步中的fltk-dist-1.3.8文件夹整体复制到fltk-demo目录中
2. 项目根目录创建src文件夹，并在其中创建main.cpp：

```c++
#include <iostream>
#include <FL/Fl.H>
#include <FL/Fl_Window.H>
#include <FL/Fl_Box.H>

int main (int argc, char *argv[]) {
    Fl_Window *window;
    Fl_Box *box;
    window = new Fl_Window(300, 180);
    window->label("HelloWorld!");
    box = new Fl_Box(20, 40, 260, 100, "Hello World!");
    box->box(FL_UP_BOX);
    box->labelsize(36);
    box->labelfont(FL_BOLD + FL_ITALIC);
    (FL_SHADOW_LABEL);
    window->end();
    window->show(argc, argv);
    return Fl::run();
}
```

3. 创建一个CMakeLists.txt内容：

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.23)
PROJECT(fltk_demo)

SET(CMAKE_CXX_STANDARD 11)

# 可执行程序名称，下面统一使用
SET(my_app_name fltk_demo)

# 指定头文件查找目录
INCLUDE_DIRECTORIES("${CMAKE_CURRENT_SOURCE_DIR}/fltk-dist-1.3.8/include")

# 将src下面的所有头文件路径保存至 all_head_files 数组变量中
# 将src下面的所有源文件路径保存至 all_source_files 数组变量中
FILE(GLOB_RECURSE all_source_files "src/*.cpp" "src/*.c")
FILE(GLOB_RECURSE all_head_files "src/*.hpp" "src*.h")

# 指定库文件查找路径，根据操作系统区分：
IF (CMAKE_SYSTEM_NAME MATCHES "Windows")
    # Windows 操作系统，需要查找 Windows-release
    MESSAGE(STATUS "current platform: Windows")
    LINK_DIRECTORIES("${CMAKE_CURRENT_SOURCE_DIR}/fltk-dist-1.3.8/lib/Windows-release")
    ADD_EXECUTABLE(
            ${my_app_name}
            WIN32 # Windows 非命令行程序
            ${all_head_files}
            ${all_source_files}
    )
    TARGET_LINK_LIBRARIES(
            ${my_app_name}
            PRIVATE
            fltk # 链接的静态库名称，这里只需要写fltk，在运行时自动查找.a/.lib
    )
ELSEIF (CMAKE_SYSTEM_NAME MATCHES "Darwin")
    # macOS 操作系统，查找 macOS-release
    MESSAGE(STATUS "current platform: macOS")
    LINK_DIRECTORIES("${CMAKE_CURRENT_SOURCE_DIR}/fltk-dist-1.3.8/lib/macOS-release")
    ADD_EXECUTABLE(
            ${my_app_name}
            ${all_head_files}
            ${all_source_files}
    )
    TARGET_LINK_LIBRARIES(
            ${my_app_name}
            PRIVATE
            fltk # 链接的静态库名称，这里只需要写fltk，在运行时自动查找.a/.lib
            # macOS 该选项必填
            "-framework Cocoa"
    )
ELSEIF (CMAKE_SYSTEM_NAME MATCHES "Linux")
    # Linux 操作系统，查找 Linux-release
    MESSAGE(STATUS "current platform: Linux ")

    LINK_DIRECTORIES("${CMAKE_CURRENT_SOURCE_DIR}/fltk-dist-1.3.8/lib/Linux-release")
    ADD_EXECUTABLE(
            ${my_app_name}
            ${all_head_files}
            ${all_source_files}
    )
    TARGET_LINK_LIBRARIES(
            ${my_app_name}
            PRIVATE
            fltk # 链接的静态库名称，这里只需要写fltk，在运行时自动查找.a/.lib
            X11 # Linux环境需要指定X11以及dl两个库才能正常显示
            dl
    )
ELSE ()
    # 其他操作系统
    MESSAGE(STATUS "other platform: ${CMAKE_SYSTEM_NAME}")
ENDIF ()
```

对于CMake的配置，我们针对不同的操作系统，我们从dist中指定的操作系统的目录查找静态库文件。此外，还有一些需要注意的：

**Windows**

Windows操作系统中，请在ADD_EXECUTABLE的应用名称后面添加WIN32，否则部分Windows操作系统窗口显示的时候，还会有一个命令行界面显示出来。

**macOS**

macOS操作系统中，请在TARGET_LINK_LIBRARIES最后添加参数：`"-framework Cocoa"`，否则fltk链接过程会有如下报错：

```
[1/1] Linking CXX executable fltk_demo
FAILED: fltk_demo 
: && 
... ...
Undefined symbols for architecture x86_64:
... ...
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
ninja: build stopped: subcommand failed.
```

**Linux**

对于Linux操作系统，由于桌面窗体程序是基于X11进行的，所以实际运行的过程中是依赖X11库的。所以，我们还需要将X11的动态库也链接到咱们程序。我们的Demo中的CMakeLists.txt针对Linux系统，如果不进行特殊处理，会出现如下类似的错误：

*undefined reference to `XGetDefault'等类似问题*

```
[ 50%] Building CXX object CMakeFiles/fltk_demo.dir/src/main.cpp.o
[100%] Linking CXX executable fltk_demo
/usr/bin/ld: /home/w4ngzhen/Desktop/fltk-demo/fltk-dist-1.3.8/lib/Linux-release/libfltk.a(Fl_get_system_colors.o): in function `getsyscolor(char const*, char const*, char const*, char const*, void (*)(unsigned char, unsigned char, unsigned char))':
Fl_get_system_colors.cxx:(.text._ZL11getsyscolorPKcS0_S0_S0_PFvhhhE+0x24): undefined reference to `XGetDefault'
/usr/bin/ld: Fl_get_system_colors.cxx:(.text._ZL11getsyscolorPKcS0_S0_S0_PFvhhhE+0x47): undefined reference to `XParseColor'
... ...
Fl_Double_Window.cxx:(.text._ZN16Fl_Double_Window4hideEv+0x21): undefined reference to `XFreePixmap'
collect2: error: ld returned 1 exit status
make[2]: *** [CMakeFiles/fltk_demo.dir/build.make:97: fltk_demo] Error 1
make[1]: *** [CMakeFiles/Makefile2:83: CMakeFiles/fltk_demo.dir/all] Error 2
make: *** [Makefile:91: all] Error 2
```

所以，我们需要进行配置，在链接阶段，链接X11动态链接库。

此外，除了X11库以外，我们还需要一个`dl`库。不配置则会有如下类似错误：

*undefined reference to symbol 'dlsym@@GLIBC_2.2.5'*

```
[ 50%] Building CXX object CMakeFiles/fltk_demo.dir/src/main.cpp.o
[100%] Linking CXX executable fltk_demo
/usr/bin/ld: /home/w4ngzhen/Desktop/fltk-demo/fltk-dist-1.3.8/lib/Linux-release/libfltk.a(Fl_Window_shape.o): undefined reference to symbol 'dlsym@@GLIBC_2.2.5'
/usr/bin/ld: /lib/x86_64-linux-gnu/libdl.so.2: error adding symbols: DSO missing from command line
collect2: error: ld returned 1 exit status
make[2]: *** [CMakeFiles/fltk_demo.dir/build.make:97: fltk_demo] Error 1
make[1]: *** [CMakeFiles/Makefile2:83: CMakeFiles/fltk_demo.dir/all] Error 2
make: *** [Makefile:91: all] Error 2
```

综合来说，基于Linux环境下的CMakeLists.txt基础配置，和其他平台的差异体现在要增加额外的两个库：

```diff
    TARGET_LINK_LIBRARIES(
            ${my_app_name}
            PRIVATE
            fltk # 链接的静态库名称，这里只需要写fltk，在运行时自动查找.a/.lib
+            X11 # Linux环境需要指定X11以及dl两个库才能正常显示
+            dl
    )
```

# 效果演示

下图是本人分别在Windows、macOS以及Linux环境下的运行效果：

![010-fltk-demo-in-all](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-19/010-fltk-demo-in-all.png)

# 附录

本文项目代码已经提交至Github

[w4ngzhen/fltk-demo (github.com)](https://github.com/w4ngzhen/fltk-demo)







