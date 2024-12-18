---
title: 使用CEF（五）— 在QT中集成CEF（2）基于CLion与CMake搭建环境
date: 2022-09-12
tags:
 - qt
 - cef
 - cmake
categories:
  - 技术
  - 使用CEF
---

在前文《使用CEF（四）— 在QT中集成CEF（1）：基本集成》中，我们使用VS+QT的插件搭建了一个基于QT+CEF的项目。时过境迁，笔者目前用的最多的就是CLion+CMake搭建C/C++项目，并且CLion提供了对C/C++强大的开发环境。此外，也想将CMake搭建QT项目作为一次实践，故由此文。

<!-- more -->

# 基础环境

- QT 5.14.2
- CEF 105.3.33以及对应版本wrapper（**特别注意，wrapper以动态库（MD）版本进行编译**。为了方便更多的开发者了解如何编译，我做了一个[视频](https://www.bilibili.com/video/BV1GV4y1u7KM)，视频是MT版本，请读者自行修改配置。）
- CMake 3.24-rc5
- VS2019

# 工程搭建

创建`QtCefCMakeDemo`文件夹，将基础环境提到的CEF的wrapper编译产物（`libcef_dll_wrapper`）+CEF相关库文件（`libcef`）、资源文件（`*.pak`）放置于`QtCefCMakeDemo/CefFiles`中：

```
QtCefCMakeDemo
 └─ CefFiles
    ├─bin
    │  ├─Debug
    │  │  │  ...
    │  │  │  libcef.dll
    │  │  │  libcef.lib
    │  │  │  libcef_dll_wrapper.lib
    │  │  │  ...
    │  │  │
    │  │  └─swiftshader
    │  │          ...
    │  │
    │  └─Release
    │      │  ...
    │      │  libcef.dll
    │      │  libcef.lib
    │      │  libcef_dll_wrapper.lib
    │      │  ...
    │      │  
    │      └─swiftshader
    │              ...
    │
    ├─include
    │  各种.h头文件
    │  ...
    └─Resources
        │  cef.pak
        │  ..
        └─locales
                ...
                zh-CN.pak
                zh-TW.pak
```

并且在`QtCefCMakeDemo`目录下创建一个`src`目录，用以存放cpp代码。将咱们在《在QT中集成CEF（1）》中编写的相关代码存放于该目录下（[QtCefDemo/QtCefDemo at main · w4ngzhen/QtCefDemo (github.com)](https://github.com/w4ngzhen/QtCefDemo/tree/main/QtCefDemo)）：

```
QtCefCMakeDemo
 ├─ CefFiles
 └─ src
      app.manifest
      main.cpp
      qtcefwindow.cpp
      qtcefwindow.h
      qtcefwindow.qrc
      qtcefwindow.ui
      simple_app.cpp
      simple_app.h
      simple_handler.cpp
      simple_handler.h
      stdafx.cpp
      stdafx.h
  ... ...
```

**请注意，这份代码已经已经有些许过时了，该份代码是基于cef_binary_87.1.13版本，而我们本文是基于cef_binary_105.3.33。所以使用新的cef、cef wrapper，但使用旧的应用层代码，势必会有问题。但是我们目前先不处理，后文会逐一列举并修改。**

## CMakeLists.txt

使用CMake来搭建QT+CEF项目，最核心的就是CMakeLists.txt文件内容：

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)

PROJECT(QtCefCMakeDemo LANGUAGES CXX)
SET(CMAKE_BUILD_TYPE DEBUG)
SET(CMAKE_CXX_STANDARD 11)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_INCLUDE_CURRENT_DIR ON)

# 【QT】CMAKE_PREFIX_PATH 实际值为本地安装的QT中的对应编译环境的目录
SET(CMAKE_PREFIX_PATH "D:\\Programs\\Qt\\Qt5.14.2\\5.14.2\\msvc2017_64")
# 配置了上述后，可以通过find_package来查找QT相关的cmake文件

# 【QT】UIC、MOC、RCC启用
# 引入的QT模块则会对.ui文件、.qtc文件以及QT中的元信息机制自动进行处理
SET(CMAKE_AUTOUIC ON)
SET(CMAKE_AUTOMOC ON)
SET(CMAKE_AUTORCC ON)

# 【QT】通过FIND_PACKAGE，CMake会查找QT相关模块cmake文件，
# 这些cmake文件自动处理了头文件的查找等，
# 不需要像配置CEF的头文件查找一样来配置QT的头文件引入
FIND_PACKAGE(Qt5 COMPONENTS Widgets REQUIRED)
# 【CEF】CEF相关头文件的引入
INCLUDE_DIRECTORIES("${CMAKE_SOURCE_DIR}/CefFiles")
INCLUDE_DIRECTORIES("${CMAKE_SOURCE_DIR}/CefFiles/include")

# 添加项目所有的文件：
# 头文件、源文件、ui文件、qrc资源文件
# 特别的，在Windows下VS下，还需要manifest文件，并且该文件在cmake3.4以后就能够自动是被并被引入
ADD_EXECUTABLE(qt-cef
        WIN32
        src/qtcefwindow.h
        src/simple_app.h
        src/simple_handler.h
        src/main.cpp
        src/qtcefwindow.cpp
        src/simple_app.cpp
        src/simple_handler.cpp
        src/qtcefwindow.ui
        src/qtcefwindow.qrc
        src/app.manifest
        )

# QT库链接
TARGET_LINK_LIBRARIES(qt-cef
        PRIVATE
        # 【QT】QT库链接
        Qt5::Widgets
        # 【CEF】cef相关库链接
        "${CMAKE_SOURCE_DIR}/CefFiles/bin/Debug/libcef.lib"
        "${CMAKE_SOURCE_DIR}/CefFiles/bin/Debug/libcef_dll_wrapper.lib"
        )
```

CMake的基础配置请各位读者自行了解。关于QT的配置，我都在CMakeLists.txt中以`【QT】`标识出；关于CEF的配置部分，我都在配置文件中以`【CEF】`标识出。

# 异常处理

此时，我们尝试编译整个项目的时候，会发现有一些编译/链接的错误，相关的错误大多数来源于CEF的头文件升级，接下来我将一一列举并处理。

## error C3646: “OVERRIDE”: 未知重写说明符

出现点：simple_app.h、simple_handler.h

原因以及解决方案：实际上在87版本中这个`OVERRIDE`是一个宏，指代的就是关键字：`override`，不过在105版本中已经不存在了，所以手动修改为c++标准关键词即可。所以解决方案就是将所有出现`OVERRIDE`的地方改为关键词`override`。

## error C2039: "Bind": 不是 "base" 的成员

出现点：simple_handler.cpp

原因以及解决方案：cef团队移除了该API（[Remove deprecated base::Bind APIs (see issue #3140)](https://github.com/chromiumembedded/cef/commit/07bc800f0080f62e0d299d3e1a878510085bebc3)），而是要求使用BindOnce，且该BindOnce所在定义的头文件由原来的`#include "include/base/cef_bind.h"`变为`#include "include/base/cef_callback.h"`。所以解决方案就是将头文件`include/base/cef_bind.h`改为引入`include/base/cef_callback.h`，且将base::Bind改为base::BindOnce。

## warning C4819: 该文件包含不能在当前代码页(936)中表示的字符。请将该文件保存为 Unicode 格式以防止数据丢失

出现点：只要不是UTF-8 with BOM的文件，都可能出现这个警告

原因以及解决方案：CLion 默认使用 UTF-8 编码，MSVC 除非明确指定否则就使用 UTF-8 with BOM 或者当前代码页（详情可以参考这篇博文：[解决 CLion + MSVC 下的字符编码问题)](https://www.cnblogs.com/Chary/p/13608011.html)），所以在CMakeLists.txt中，**在ADD_EXECUTABLE之前加上**：

```cmake
# 解决warning C4819
ADD_COMPILE_OPTIONS("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
ADD_COMPILE_OPTIONS("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
```

## error C2664: “void CefWindowInfo::SetAsChild(HWND,const CefRect &)”: 无法将参数 2 从“RECT”转换为“const CefRect &”

出现点：qtcefwindow.cpp

原因以及解决方案：void CefWindowInfo::SetAsChild(HWND parent, const CefRect &windowBounds)，第二个参数类型由原来的windef.h中定义的RECT结构体，调整为`CefRect`类，即：

```diff
-    RECT win_rect;
     QRect rect = this->geometry();
-    win_rect.left = rect.left();
-    win_rect.right = rect.right();
-    win_rect.top = rect.top();
-    win_rect.bottom = rect.bottom();
+// CEF引入CefRect，而不是windef.h中的RECT
+    CefRect win_rect(
+            rect.left(),
+            rect.top(),
+            rect.left() + rect.width() * devicePixelRatio(),
+            rect.top() + rect.height() * devicePixelRatio());
```

## libcef_dll_wrapper.lib(libcef_dll_wrapper.obj) : error LNK2038: 检测到“_ITERATOR_DEBUG_LEVEL”的不匹配项: 值“0”不匹配值“2”(mocs_compilation.cpp.obj 中)

出现点：链接阶段错误

原因以及解决方案：针对该问题，首先通过网上搜寻的博文了解到是：`当前工程是Debug版本，而引用的库文件时Release版本`。排查libcef_dll_wrapper.lib，确实使用的Debug版本。从报错了解到与`mocs_compilation.cpp.obj`的`_ITERATOR_DEBUG_LEVEL`不一致。但是，这个`mocs_compilation.cpp.obj`是通过咱们项目生成的，是QT的MetaObject元对象机制下，MOC参与代码生成、编译输出的，其自动生成的代码在`cmake-build-debug`目录下的`qt-cef_autogen`中：

![010-autogen-in-cmake](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/010-autogen-in-cmake.png)

该cpp编译单元编译后的产物在`项目根目录/cmake-build-debug/CMakeFiles/qt-cef.dir/qt-cef_autogen`下：

![020-autogen-obj-in-cmake](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/020-autogen-obj-in-cmake.png)

使用VS的工具（ [适用于开发人员的命令行 shell 和提示 - Visual Studio (Windows) | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022)）中的**dumpbin.exe**工具（[DUMPBIN 参考 | Microsoft Docs](https://docs.microsoft.com/zh-cn/cpp/build/reference/dumpbin-reference?view=msvc-170)），可以查看库文件的`_ITERATOR_DEBUG_LEVEL`值。操作方式为：

1. 找到VS开发者工具，方式有几种，主要有：1、[从 Windows 菜单中启动](https://docs.microsoft.com/zh-cn/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022#start-from-windows-menu)；2、[从文件菜单启动](https://docs.microsoft.com/zh-cn/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022#start-from-file-browser)；
1. 启动后进入命令行，执行命令：

```bash
dumpbin /directives "库文件路径"
```

mocs_compilation.cpp.obj的_ITERATOR_DEBUG_LEVEL值

![030-dumpbin-mocs-obj](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/030-dumpbin-mocs-obj.png)

libcef_dll_wrapper.lib中一些obj的_ITERATOR_DEBUG_LEVEL值：

![040-dumpbin-libcef_dll_wrapper](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/040-dumpbin-libcef_dll_wrapper.png)

可以看出，两份库代码确实是不一样的。由于libcef_dll_wrapper.lib我们已经完成了编译，这里我们不考虑重新编译该lib库，而是通过配置CMake，让生成的mocs_compilation.cpp.obj等obj的_ITERATOR_DEBUG_LEVEL值为0，来匹配libcef_dll_wrapper.lib。所以，解决方案就是在CMakeLists.txt中，添加配置（[c++ - How to add _ITERATOR_DEBUG_LEVEL to CMake? - Stack Overflow](https://stackoverflow.com/questions/54246427/how-to-add-iterator-debug-level-to-cmake)）：

```diff
# 解决warning C4819，需要在ADD_EXECUTABLE前加上
ADD_COMPILE_OPTIONS("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
ADD_COMPILE_OPTIONS("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
+# 控制项目所有编译单元_ITERATOR_DEBUG_LEVEL的值，
+# 这里设置为和libcef_dll_wrapper.lib中的obj一致。
+ADD_COMPILE_DEFINITIONS($<$<CONFIG:Debug>:_ITERATOR_DEBUG_LEVEL=0>)

# 【QT】CMAKE_PREFIX_PATH 实际值为本地安装的QT中的对应编译环境的目录
SET(CMAKE_PREFIX_PATH "D:\\Programs\\Qt\\Qt5.14.2\\5.14.2\\msvc2017_64")
```

不出意外，此时我们已经处理了所有的编译和链接过程中的问题。控制台会显示：

```
====================[ Build | qt-cef | Debug ]==================================
D:\Programs\ToolBoxApp\apps\CLion\ch-0\222.3739.54\bin\cmake\win\bin\cmake.exe --build D:\Projects\cpp-projects\qt-projects\qt-cef\QtCefCMakeDemo\cmake-build-debug --target qt-cef -j 12
[1/8] Automatic MOC and UIC for target qt-cef
[2/8] Building CXX object CMakeFiles\qt-cef.dir\qt-cef_autogen\UVLADIE3JM\qrc_qtcefwindow.cpp.obj
[3/8] Building CXX object CMakeFiles\qt-cef.dir\src\simple_app.cpp.obj
[4/8] Building CXX object CMakeFiles\qt-cef.dir\src\simple_handler.cpp.obj
[5/8] Building CXX object CMakeFiles\qt-cef.dir\qt-cef_autogen\mocs_compilation.cpp.obj
[6/8] Building CXX object CMakeFiles\qt-cef.dir\src\qtcefwindow.cpp.obj
[7/8] Building CXX object CMakeFiles\qt-cef.dir\src\main.cpp.obj
[8/8] Linking CXX executable qt-cef.exe

Build finished
```

但是在运行的过程中理论山还会出现两个问题：

## Process finished with exit code -1073740791 (0xC0000409)

出现这个问题的时候，使用CLion的Debug模式进行，会看到错误调用栈：

![050-debug-string-error](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/050-debug-string-error.png)

经过问题排查，主要原因点：

![060-string-error-detail](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/060-string-error-detail.png)

在qtcefwindow构造函数中调用`CefBrowserHost::CreateBrowser`API，会传入初始要打开的页面地址，然而QString.toStdString得到string有问题（后续排查具体原因）。解决方案就是直接使用std::string变量即可：

```diff
     // 以下是将 SimpleHandler 与窗体进行关联的代码
     CefWindowInfo cef_wnd_info;
-    QString str_url = "https://www.cnblogs.com/w4ngzhen";
+    std::string str_url = "https://www.cnblogs.com/w4ngzhen";
     QRect rect = this->geometry();
     CefRect win_rect(
             rect.left(),
@@ -25,7 +25,7 @@ QtCefWindow::QtCefWindow(QWidget* parent)
     simple_handler_ = CefRefPtr<SimpleHandler>(new SimpleHandler());
     CefBrowserHost::CreateBrowser(cef_wnd_info,
         simple_handler_,
-        str_url.toStdString(),
+        str_url,
         cef_browser_settings,
         nullptr,

```

## "Invalid COM thread model change" 或 运行后异常退出报错Exception 0x80000003 encountered at address 0x7ffbc43e9f3c

解决掉上述问题以后，笔者的环境下还会出现两种类似的问题：

1. "Invalid COM thread model change"（实际上有些同学机器上，这个问题先于上面的字符串问题）
2.  运行后异常退出报错Exception 0x80000003 encountered at address 0x7ffbc43e9f3c

这两种情况都是一个解决方案。问题点在于，QT的事件循环在多个进程（浏览器进程、渲染进程）均被初始化。实际上只需要在浏览器进程即可。解决方案就是将main.cpp中的init_for_cef提取到最前：

```diff
-    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);  // 解决高DPI下，界面比例问题
-    QApplication a(argc, argv);
+    // 将init_qt_cef提取到QApplication初始化之前
+    // 对于CEF多进程架构模型
+    // 因为【渲染进程】启动后，init_qt_cef中执行的CefExecuteProcess会阻塞住，
+    // 如果在此之前启动了QT的事件循环，那么会导致QT出现异常
+    // 所以，我们将init_qt_cef提前到QApplication初始化之前，
+    // 保证无论是浏览器进程还是渲染进程启动，都会进入init_qt_cef，但渲染进程会在里面阻塞，
+    // 不会进入后续的QT应用初始化
     const int result = init_qt_cef(argc, argv);
     if (result >= 0)
     {
         return result;
     }

+    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);  // 解决高DPI下，界面比例问题
+    QApplication a(argc, argv);
+
     QtCefWindow w;
     w.show();
     a.exec();
```

对于CEF多进程架构模型，因为**渲染进程**启动后，init_qt_cef中执行的CefExecuteProcess会阻塞住，如果在此之前启动了QT的事件循环，那么会导致QT出现异常。 所以，我们将init_qt_cef提前到QApplication初始化之前，保证无论是**浏览器进程**还是**渲染进程**启动后，都会进入init_qt_cef，但渲染进程会在里面阻塞，不会进入后续的QT应用初始化。

# 效果演示与代码库

![070-show](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/070-show.gif)

与本文相关的代码已经提交至Github，且按照整个文章的编写流程进行提交：

[w4ngzhen/QtCefCmakeDemo (github.com)](https://github.com/w4ngzhen/QtCefCmakeDemo)

![080-git-commits](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-12/080-git-commits.png)

