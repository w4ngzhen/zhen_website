---
title: 使用CEF（六）— 解读CEF的cmake工程配置
date: 2023-10-11
tags:
 - cef
 - cmake
categories:
  - 技术
  - 使用CEF
---

距离笔者的《使用CEF》系列的第一篇文章居然已经过去两年了，在这么长一段时间里，笔者也写了很多其它的文章，再回看《使用CEF（一）— 起步》编写的内容，文笔稚嫩，内容单薄是显而易见的（主要是教大家按部就班的编译libcef_dll_wrapper库文件）。笔者一直以来的个性就是希望自己学习到的知识，研究出的内容，踩过的坑能够及时的写出来，介绍给更多的小伙伴。

这篇文章产生的背景是最近笔者再一次仔细的阅读了CEF binary distribution（CEF二进制分发包）的工程代码以及根目录下的CMakeLists.txt文件的所了解到的东西，希望在本文能够让读者小伙伴对于CEF binary distribution的工程结构有一个较为清晰的了解。

<!-- more -->

# CMake基础导入

CMake是什么，它和Unix下的make+gcc、macOS下的xcode+clang以及Windows下的VS+msvc工具链的关系不在本文解释，但阅读本文还是需要对CMake所扮演的角色有基本认识，所以如果你还不是特别清楚，建议先从笔者这一篇文章了解下《[C与CPP常见编译工具链与构建系统简介 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/655701220)》。CMake本身无法构建任何的应用，它生成不同构建工具所需要的配置或某种输入，再让构建工具基于配置调用工具链，对代码进行构建。

## target

一般来说，我们使用CMake来构建某种产物（这里的“构建”不严谨，只是方便描述），这个产物可以是可执行二进制程序直接运行，可以是一个库文件。构建产物在CMake中被抽象成了名为**target**的东西，CMake的核心运转就是围绕target进行的。

在CMake中定义某个target，最最最基础的方式有两种：`add_executable`和`add_library`。

**add_executable()**：[add_executable — CMake 3.27.6 Documentation](https://cmake.org/cmake/help/latest/command/add_executable.html)

该命令用于定义一个可以构建成可执行程序的target，简单用法形式如下：

```cmake
add_executable(<name> [WIN32] [MACOSX_BUNDLE]
               [EXCLUDE_FROM_ALL]
               [source1] [source2 ...])
```

- 第一个必填参数`name`，就是我们要编译的可执行程序target的名称；
- 可选参数`WIN32`、`MACOS_BUNDLE`。分别用于Windows平台和macOS平台**可执行应用程序**的构建。如果没有配置这个参数，你会发现最终编译的可执行程序本质上就是从控制台程序启动的程序。两个最直观的例子：在Windows上的QT GUI项目，没有配置`WIN32`参数，那么编译后运行起来时除了我们的GUI窗体展示，还会有一个黑色控制台窗口展示；在macOS上，你经常看到的某某应用`XXX.app`实际上是一个bundle，里面有这个应用的各种配置、实际运行的可执行文件等，如果你想要最终构建出来的属于这种应用程序，那么就需要`MACOS_BUNDLE`参数；
- 可选参数`EXCLUDE_FROM_ALL`，表明整个项目构建的时候，排除当前这个target；
- 至于`source1`、`source2`等等就是头文件、源码文件了。

**add_library()**：[add_library — CMake 3.27.6 Documentation](https://cmake.org/cmake/help/latest/command/add_library.html)

该命令用于定义一个生成的库文件的target，**普通用法**形式如下：

```cmake
add_library(<name> [STATIC | SHARED | MODULE]
            [EXCLUDE_FROM_ALL]
            [<source>...])
```

- 第一个必填参数`name`，就是我们要编译的库文件target的名称；
- 参数`STATIC`、`SHARED`、`MODULE`互斥三选一。`STATIC`表明希望将这个库文件编译为静态库；`SHARED`表明希望将这个库文件编译为动态连接库；`MODULE`表明编译为一个动态库，但是通过运行时以程序的方式加载（比如dlopen在Unix-like系统中，或LoadLibrary在Windows系统中）。
- 可选参数`EXCLUDE_FROM_ALL`，表明整个项目构建的时候，排除当前这个target；
- 至于`source`等等就是头文件、源码文件了。

**target_include_directories与target_link_libraries**

想要构建C/C++工程，我们经常需要在编译阶段使用**外部库**的头文件分析依赖与内存布局，以及在链接阶段链接这些外部库文件。在CMake中，我们一般使用`target_include_directories`指令来指定对应target编译过程中外部库头文件的搜索路径，以及使用`target_link_libraries`指令来指定链接阶段要链接哪些库文件，具体用法读者自行了解。

值得注意的是，除了上述两个指令外，你还会搜索到两个类似的指令`include_directories`和`link_libraries`。这两个指令命名上没有`"target_"`前缀，其作用主要是提供全局的头文件和链接库搜索路径。这个两个全局作用的指令的背景在于CMake是支持多target模块构建的，可以通过项目顶层的CMakeLists.txt中配置这两个指令，让子模块target共享这些头文件和库文件路径配置。但是如没有必要，尽可能使用`target_xxx`来给指定的target配置。举一反三，CMake中还有很多的`target_`开头的指令，其目的都是针对某个指定的target的配置。

由于篇幅原因，本文关于CMake基本导入的部分就介绍到这里，接下来，让我们逐步分析CEF binary distribution中的CMakeLists.txt。

# 顶层CMakeLists.txt

![010-root-CMakeLists](https://res.zhen.wang/images/post/2023-10-11/010-root-CMakeLists.png)

## OVERVIEW

overview部分简单介绍了CMake，然后介绍CEF binary distribution不同平台下支持的项目构建系统和工具链：

```cmake
# Linux:      Ninja, GCC 7.5.0+, Unix Makefiles
# MacOS:      Ninja, Xcode 12.2 to 13.0
# Windows:    Ninja, Visual Studio 2022
```

## CMAKE STRUCTURE

该部分介绍了CEF binary distribution的CMAKE工程结构，说明了CEF二进制分发包主要由以下几个部分组成：

```cmake
# CMakeLists.txt              Bootstrap that sets up the CMake environment.
# cmake/*.cmake               CEF configuration files shared by all targets.
# libcef_dll/CMakeLists.txt   Defines the libcef_dll_wrapper target.
# tests/*/CMakeLists.txt      Defines the test application target.
```

- CMakeLists.txt：组织构建CEF二进制分发的CMake环境。
- cmake/*.cmake：CMake配置文件，可被所有的target使用。
- libcef_dll/CMakeLists.txt：定义了`libcef_dll_wrapper`这个target的CMake配置。
- tests/*/CMakeLists.txt：定义了所有的测试Demo应用target。

## BUILD REQUIREMENTS

该部分主要介绍了编译`libcef_dll_wrapper`以及相关样例demo在不同操作系统平台上的环境要求。

## BUILD EXAMPLES

这一部分主要介绍了如何构建`libcef_dll_wrapper`以及demo。具体的做法就是在`cef_binary_xxx目录`（后续都用该指代CEF binary distribution文件夹根目录）中创建一个名为`build`的目录，进入该目录后，针对不同的平台，使用CMake生成不同的构建系统的工程配置，并进行构建。其中，由于Ninja是一个跨平台的**构建系统**，所以你会看每个平台都有Ninja构建系统的生成指令。例如，下图展示了在macOS x86 64位架构上使用CMake生成对应的构建方案的两种方式：1、xcode构建方案（xcodebuild构建方案体系）；2、Ninja构建方案。

![020-macOS-cmake-build](https://res.zhen.wang/images/post/2023-10-11/020-macOS-cmake-build.png)

> 无论是xcode还是ninja，都是构建系统，在macOS上最终调用编译工具链是底层的clang/LLVM。

再比如，在Windows64位系统上也有两种方式：1、VisualStudio解决方案（MSBuild构建方案体系）；2、Ninja构建方案。

![030-Windows-cmake-build](https://res.zhen.wang/images/post/2023-10-11/030-Windows-cmake-build.png)

> 同样的，无论是vs MSBuild还是ninja，都是构建系统，在Windows上最终调用的是底层的msvc编译工具链。

对于使用Ninja，读者会看到都会调用`ninja cefclient cefsimple`，这个命令运行后，会编译demo中的cefclient和cefsimple两个项目，这里只是官方例子，在实际使用过程中，并不是一定要按照它的操作来。另外，有读者可能有疑问，这个过程并没有看到关于`libcef_dll_wrapper`项目的构建，这里先提前说明一下，在cefsimple和cefclient等demo中依赖了`libcef_dll_wrapper`并通过配置进行了指定，所以构建的过程中，会优先自动编译`libcef_dll_wrapper`。关于这块，等我们后面详解的时候会介绍的。

在看完了关于不同平台的构建方式以后，我们往下会看到关于`"Global setup."`的部分。这一部分开始，就是CMake真正有关的部分了。让我们首先删除掉所有的注释，逐步分析这个顶层CMakeLists.txt的配置：

![040-root-CMakeLists](https://res.zhen.wang/images/post/2023-10-11/040-root-CMakeLists.png)

剔除了注释以后，会发现其实内容并不多。这里我们首先从上图第8行开始关于设置`CEF_ROOT`和`CMAKE_MODULE_PATH`的分析：配置首先定义了`CEF_ROOT`，它使用了CMake提供的变量`CMAKE_CURRENT_SOURCE_DIR`，也就是当前CMakeLists.txt所在目录：`cef_binary_xxx`目录；然后对`CMAKE_MODULE_PATH`**追加**了`${CEF_ROOT}/cmake`这个目录。

之所以这样做，是因为接下来`find_package(CEF REQUIRED)`会根据`CMAKE_MODULE_PATH`所提供的路径进行搜索。关于`find_package`，网上解析的文章很多，这里只简单说明下，CMake官方文档中提到`find_package`有两种搜索模式，其中一种就是**模块搜索模式**（Module mode），该搜索模式说明如下：

> Module mode
In this mode, CMake searches for a file called `Find<PackageName>.cmake`, looking first in the locations listed in the `CMAKE_MODULE_PATH`, then among the Find Modules provided by the CMake installation. If the file is found, it is read and processed by CMake. 

翻译过来就是：当运行`find_package(PackageName)`的时候，CMake会在`CMAKE_MODULE_PATH`路径列表中，查找名为`Find<PakcageName>.cmake`文件，找到后就会对该配置文件加载并处理。对照本例，`find_package(CEF REQUIRED)`，在模块搜索模式下，则是需要查找一个名为`FindCEF.cmake`的文件。由于我们在`CMAKE_MODULE_PATH`中追加了`${CEF_ROOT}/cmake`这个目录，即`cef_binary_xxx/cmake`目录，所以CMake会搜索这个目录，该目录确实存在`FindCEF.cmake`文件，于是被CMake命中并加载了。那么，接下来让我们打开该`FindCEF.cmake`文件，一探究竟。

### FindCEF.cmake

![050-FindCEF](https://res.zhen.wang/images/post/2023-10-11/050-FindCEF.png)

`FindCEF.cmake`很好理解，大致处理过程是：

首先从CMake全局上下文或系统环境变量等地方读取名为`CEF_ROOT`的值，这个值是一个目录，指代了`cef_binary_xxx`目录，然后校验该目录路径是否合法（路径下的cmake目录是否存在），并赋值给`_CEF_ROOT`（**这个值很关键，接下来都是使用这个_CEF_ROOT值**）；

然后，给`CMAKE_MODULE_PATH`追加`${_CEF_ROOT}/cmake`路径，与之前`cef_binary_xxx/CMakeList.txt`中追加该PATH目的不一样，这一次追加`CMAKE_MODULE_PATH`值的核心目的是为下面调用`include("cef_variables")`和`include("cef_macros")`的时候，能够找到`${_CEF_ROOT}/cmake`路径下名为`cef_variables.cmake`和`cef_macros.cmake`文件。

> CMake的官方文档告诉我们，CMake在处理`include("abc")`的时候，会搜索`CMAKE_MODULE_PATH`路径下名为`abc.cmake`的文件进行加载处理。[CMake - include](https://cmake.org/cmake/help/latest/command/include.html#include)

看到这里，有的读者可能已经绕晕了，我们做一个简单的流程图来描述这个过程：

cef-binary-xxx/CMakeList.txt -> find_package(CEF REQUIRED) -> 在第一次 CMAKE_MODULE_PATH路径配置前提下，找到了 FindCEF.cmake配置读取；FindCEF.cmake -> include("cef_variables")、include("cef_macros")，按顺序加载 cef_variables.cmake 和 cef_macros.cmake。

我们暂时不深入研究`cef_variables.cmake`、`cef_macros.cmake`里的内容，后面遇到一些特殊的变量、宏的时候，我们再来解释，这里我们可以先跳出细节，可以认为`cef_variables`和`cef_macros`里面分别定义了一些变量配置和宏定义，供后续CMake处理流程读取或调用。

# libcef_dll/CMakeLists.txt

现在，让我们回到`cef_binary_xxx/CMakeLists.txt`，在`find_package(CEF)`之后，紧接着的就是`add_subdirectory()`指令：

```cmake
# Include the libcef_dll_wrapper target.
# Comes from the libcef_dll/CMakeLists.txt file in the binary distribution
# directory.
add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)
```

这里出现了一个变量：`CEF_LIBCEF_DLL_WRAPPER_PATH`，它来源于`cef_variables.cmake`中定义的：

![060-CEF_LIBCEF_DLL_WRAPPER_PATH](https://res.zhen.wang/images/post/2023-10-11/060-CEF_LIBCEF_DLL_WRAPPER_PATH.png)

也就是说，在本例中，`add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)`就是添加了子目录`cef_binary_xxx/libcef_dll`。一旦添加了该子模块目录，CMake就会在该目录下搜索对应的CMakeLists.txt文件并进行加载（这里就是`cef_binary_xxx/libcef_dll/CMakeLists.txt`）。

这份`libcef_dll/CMakeLists.txt`主要就是将`libcef_dll_wrapper`的各种源码、以及`libcef`的头文件、各种平台特定的源代码文件放到一些CMake变量中，最后的通过`add_library`指令，定义了一个名为`libcef_dll_wrapper`的target，并将前面的源代码、头文件等添加到这个target中：

![070-add-source-to-target](https://res.zhen.wang/images/post/2023-10-11/070-add-source-to-target.png)

写到这里，我们可以对`cef_binary_xxx/CMakeLists.txt`文件做一个简单的概念总结。首先，该CMakeLists.txt扮演的是项目顶层统领全局的角色，它并没有定义过任何的target，而是通过两个步骤组织了`CEF binary distribution目录中的libcef_dll_wrapper、demo等target的构建：

步骤一：负责预构造CMake处理环境上下文，包括准备各种配置变量、宏方法等，供后续过程使用。这个过程具体是是通过加载`FindCEF.cmake`，并在该文件内部再加载`cef_variables.cmake`和`cef_macros.cmake`两个配置。

步骤二：通过`add_subdirectory`添加并管理起子模块target，包括`libcef_dll_wrapper`以及各种demo的target。这个过程CMake会读取对应路径下的`CMakeLists.txt`并加载。同时，这些文件中使用到的一些CEF提供的变量、宏都来自于步骤一所加载的`cef_variables.cmake`和`cef_macros.cmake`。

# cefsimple/CMakeLists.txt

因为`libcef_dll_wrapper`这个target最终产物是一个库文件，所以这个target所在CMakeLists.txt内容虽然很多，但是比较直白，就是各种源代码、头文件的添加。但是，如果target产物是一个可执行程序，CMakeLists.txt还会这么简单吗？这里我们分析下cefsimple这个样例的CMakeLists.txt。

首先，cefsimple存放于`cef_binary_xxx/tests/cefsimple`目录中，在`cef_binary_xxx/CMakeLists.txt`中，同样通过`add_subdirectory`添加：

```CMake
f(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/tests")
  add_subdirectory(tests/cefsimple) # cefsimple
  add_subdirectory(tests/gtest)
  add_subdirectory(tests/ceftests)
endif()
```

> 这里之所以使用一个目录判断，目测是在CEF binary distribution的Minimal最小版本中，是剔除了样例工程的，所以做了一个IF判断。

所以，接下来我们开始分析`cef_binary_xxx/tests/cefsimple/CMakeLists.txt`文件。

源文件定义

该文件实际上也分为两个部分。第一部分就是通过变量来存储cefsimple的相关源码、头文件：

![080-add-source-for-simple-demo](https://res.zhen.wang/images/post/2023-10-11/080-add-source-for-simple-demo.png)

这一块我们挑一个比较典型的处理：

![090-handle-flow](https://res.zhen.wang/images/post/2023-10-11/090-handle-flow.png)

首先使用`CEFSIMPLE_SRCS`来存储平台无关的源代码和头文件。其次，由于不同操作系统平台下有一些平台特定的源代码，例如macOS下，设置窗体标题，我们可以使用objective-c代码（`.m`/`.mm`文件）来使用原生API操作窗体标题，所以使用`CEFSIMPLE_SRCS_平台标识`变量存储这些平台特定代码的列表；最后，使用一个名为`APPEND_PLATFORM_SOURCES`的宏来处理`CEFSIMPLE_SRCS`变量，这里有两个疑问点：1、这个宏的来源和作用；2、`CEFSIMPLE_SRCS_平台标识`变量似乎没有用到。这两个疑问点一起解释。实际上，这个宏就是来源于`cef_macros.cmake`中，找到对应宏的源码：

```CMake
# Append platform specific sources to a list of sources.
macro(APPEND_PLATFORM_SOURCES name_of_list)
  if(OS_LINUX AND ${name_of_list}_LINUX)
    list(APPEND ${name_of_list} ${${name_of_list}_LINUX})
  endif()
  if(OS_POSIX AND ${name_of_list}_POSIX)
    list(APPEND ${name_of_list} ${${name_of_list}_POSIX})
  endif()
  if(OS_WINDOWS AND ${name_of_list}_WINDOWS)
    list(APPEND ${name_of_list} ${${name_of_list}_WINDOWS})
  endif()
  if(OS_MAC AND ${name_of_list}_MAC)
    list(APPEND ${name_of_list} ${${name_of_list}_MAC})
  endif()
endmacro()
```

这段宏的逻辑实际上就是通过判断操作系统平台，使用CMake提供的list APPEND机制，将入参`name_of_list`和`name_of_list_平台标识`合成为一个list列表。比较trick的就是，在调用`APPEND_PLATFORM_SOURCES(CEFSIMPLE_SRCS)`，内部比如`${${name_of_list}_MAC}` 就是`${CEFSIMPLE_SRCS_MAC}`，即获取这个变量的数据。后面剩下关于配置源文件的方式类似，这里就请读者自行分析了。

现在，让我们回到对cefsimple/CMakeLists.txt本身的分析，接下来我们分析比较重要的第二部分：可执行程序的生成：

![100-os-target](https://res.zhen.wang/images/post/2023-10-11/100-os-target.png)

这里我们对macOS平台的可执行程序生成进行讲解，因为它相对于在Windows和Linux更加复杂。首先，定义了在macOS平台下会添加一些编译指令（譬如支持objective-c语言编译）：

```cmake
option(OPTION_USE_ARC "Build with ARC (automatic Reference Counting) on macOS." ON)
if(OPTION_USE_ARC)
  list(APPEND CEF_COMPILER_FLAGS
    -fobjc-arc
    )
  set_target_properties(${target} PROPERTIES
    CLANG_ENABLE_OBJC_ARC "YES"
    )
endif()
```

然后，设置了输出的可执行程序一些名称变量，这里就是`"cefsimple.app"`：

```cmake
# Output path for the main app bundle.
set(CEF_APP "${CEF_TARGET_OUT_DIR}/${CEF_TARGET}.app")

# Variables referenced from the main Info.plist file.
set(EXECUTABLE_NAME "${CEF_TARGET}")
set(PRODUCT_NAME "${CEF_TARGET}")
```

> 再次需要提到的是，在macOS，一般可执行程序都会生成为一个App Bundle（[About Bundles (apple.com)](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/AboutBundles/AboutBundles.html)）。

如果启用了`USE_SANDBOX`标识，则会使用自定义宏（也是在之前的cef_macro.cmake中定义的）`ADD_LOGICAL_TARGET`进行特殊的处理：

```cmake
if(USE_SANDBOX)
  # Logical target used to link the cef_sandbox library.
  ADD_LOGICAL_TARGET("cef_sandbox_lib" "${CEF_SANDBOX_LIB_DEBUG}" "${CEF_SANDBOX_LIB_RELEASE}")
endif()
```

接下来就是定义核心应用：

```cmake
# Main app bundle target.
add_executable(${CEF_TARGET} MACOSX_BUNDLE ${CEFSIMPLE_RESOURCES_SRCS} ${CEFSIMPLE_SRCS})
SET_EXECUTABLE_TARGET_PROPERTIES(${CEF_TARGET})
add_dependencies(${CEF_TARGET} libcef_dll_wrapper)
target_link_libraries(${CEF_TARGET} libcef_dll_wrapper ${CEF_STANDARD_LIBS})
set_target_properties(${CEF_TARGET} PROPERTIES
  MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/mac/Info.plist
  )
```

这段代码执行逻辑解释如下：

1. 使用`add_executable`定义了主程序target，注意添加了参数`"MACOSX_BUNDLE"`表明最终生成的target是一个macOS下的App Bundle，和在Windows下的`"WIN32"`参数异曲同工；
2. 使用自定义宏`SET_EXECUTALBE_TARGET_PROPERTIES`为target添加一些属性；
3. 使用指令`add_dependencies`定义了我们当前cefsimple依赖了一个`libcef_dll_wrapper`target，**该指令的核心作用就是能够确定一个target在生成的过程中需要什么依赖。**
4. 设置了target一些特殊的properties，这里主要就是定义当生成macOS的App Bundle的时候，会在Bundle中生成Info.plist，这个文件是macOS下App Bundle中一个比较重要文件，用来定义应用的一些与macOS操作系统相关的属性，例如是否支持高分屏检测等。开发过Windows应用的小伙伴都知道，在Windows下，会有一个app.manifest文件，它俩也是异曲同工。

接下来就是使用CMake提供的`add_custom_command`指令，定义了编译生成以后（`"POST_BUILD"`标识），将相关的文件拷贝至目标目录的流程：

```cmake
# Copy the CEF framework into the Frameworks directory.
add_custom_command(
  TARGET ${CEF_TARGET}
  POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory
          "${CEF_BINARY_DIR}/Chromium Embedded Framework.framework"
          "${CEF_APP}/Contents/Frameworks/Chromium Embedded Framework.framework"
  VERBATIM
  )
```

> 在使用CMake定义项目结构的时候，我们可以通过`add_custom_command`来实现编译、构建过程中一些生命周期节点的处理逻辑，譬如拷贝依赖库等。

接下来的foreach指令，这里定义了n个helper的AppBundle target。譬如渲染进程、GPU加速进程、工具进程等具有特定功能的进程help程序：

![110-helper-target](https://res.zhen.wang/images/post/2023-10-11/110-helper-target.png)

值得注意的是，在macOS下，这里helper的`add_executable()`添加的是`CEFSIMPLE_HELPER_SRCS`，这个变量里面存储的是：

![120-helper-source](https://res.zhen.wang/images/post/2023-10-11/120-helper-source.png)

翻看该`process_helper_mac.cc`源码，其实并不复杂：

```c
// Entry point function for sub-processes.
int main(int argc, char* argv[]) {
#if defined(CEF_USE_SANDBOX)
  // Initialize the macOS sandbox for this helper process.
  CefScopedSandboxContext sandbox_context;
  if (!sandbox_context.Initialize(argc, argv)) {
    return 1;
  }
#endif

  // Load the CEF framework library at runtime instead of linking directly
  // as required by the macOS sandbox implementation.
  CefScopedLibraryLoader library_loader;
  if (!library_loader.LoadInHelper()) {
    return 1;
  }

  // Provide CEF with command-line arguments.
  CefMainArgs main_args(argc, argv);

  // Execute the sub-process.
  return CefExecuteProcess(main_args, nullptr, nullptr);
}
```

这里只要熟悉CEF的多进程架构就能理解。不熟悉的伙伴可以阅读这篇文章：[使用CEF（三）— 从CEF官方Demo源码入手解析CEF架构与CefApp、CefClient对象 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365414447)。

关于cefsimple/CMakeLists.txt剩下的内容其实也不复杂了，读者可以顺着本文的思路进一步阅读。

# 写在最后

通过顶层CMakeLists.txt的说明，不难发现，cef_binary_xxx本身既是包含了了libcef_dll_wrapper源码构建的工程，同时也是一个比较标准的，想要使用libcef+libcef_dll_wrapper的CMake工程，所以，你才会在顶层CMakeLists.txt看到官方介绍了几种基于cef_binary_xxx的CMake工程结构的项目集成案例：

![130-how-to-intergate](https://res.zhen.wang/images/post/2023-10-11/130-how-to-intergate.png)

我的博客即将同步至腾讯云开发者社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=3d9bi2yhvncwk
