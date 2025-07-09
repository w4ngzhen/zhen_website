---
title: 使用CEF（七）详解macOS下基于CEF的多进程应用程序CMake项目搭建
date: 2023-12-12
tags:
 - cef
 - macOS
 - cmake
categories:
  - 技术
  - 使用CEF
---

由于macOS下的应用程序结构导致了CEF这样的多进程架构程序在项目结构、运行架构上有很多细节需要关注，这一块的内容比起Windows要复杂的多，所以本文将会聚焦macOS下基于CEF的多进程应用架构的环境配置，并逐一说明了CMake的相关用法和CEF应用配置细节。

<!-- more -->

# 前言

在进行搭建之前，我们首先必须要弄清楚一个问题，我们最终到底要生成几个可执行应用。为什么要搞清楚这个问题呢？了解CEF的读者都知道，CEF属于多进程架构体系，包含有一个主进程管理整个浏览器应用（包括原生GUI窗体等），以及多种类型的子进程各自独立负责各自的职责（比如渲染进程以及GPU加速进程等）。

笔者在以前的文章中曾介绍过CEF中提供的样例cefsimple在Windows操作系统上的构建流程，我们发现这个cefsimple项目在编译后会最终只生成了一个exe可执行程序，而在运行时为了达到多进程的目的，该exe首先作为主进程入口启动，内部在准备启动子进程的时候，其做法是调用该exe本身，并通过命令行参数的形式来区分主进程和其他子进程。也就是说，该exe应用内部不仅包含了主进程代码，也包含了子进程代码，源代码中会根据命令行参数（`--type=xxx`）通过分支让主进程和子进程走到不同的逻辑：

![010-cef-exe-excute-flow](https://static-res.zhen.wang/images/post/2023-12-12/010-cef-exe-excute-flow.png)

而在macOS下，由于macOS本身对于应用程序的权限管理与Windows存在差异，它具备有一套特殊的沙盒机制来保证应用程序彼此独立和安全。所以，我们不建议像Windows那样最终通过编译生成一个App Bundle，来多次启动自己。一个很直观的例子可以解释这一点：假设我们现在基于CEF的应用程序编译并构建了一个App Bundle，这个app内将主进程代码和子进程代码写在了一起，通过运行时逻辑来区分。此时，假设主进程需要macOS的“钥匙串”权限，读取用户的一些配置。由于macOS权限是给到Bundle应用层面的，所以尽管我们只想让主进程得到“钥匙串”访问权限，但因为主进程和子进程都是同一个Bundle，无形中导致了子进程也同样拥有了这个权限，而像渲染进程这样的子进程，里面会运行js代码、wasm等第三方代码逻辑，一旦出现了BUG，就会存在权限泄漏风险。如果我们把主进程和子进程分离到两个Bundle，主进程所在Bundle获取某些系统权限，而渲染进程获取某些必要权限，就能做到主进程和子进程权限分离的目的，为安全性提供了一定保证。

所以，在了解了macOS下的CEF应用构建思路以后，我们开始搭建对应项目，并在搭建过程中对涉及的配置逐一解释，希望能够帮助读者理清项目脉络。

# 搭建

## 基础准备

搭建的步骤分为以下几步：

1）下载cef的二进制分发文件（`cef_binary_xxx`），将它解压存放到某个文件夹（可以不用放在项目目录下）；

2）配置一个环境变量`CEF_ROOT`，需要该环境变量值配置为`cef_binary_xxx`所在目录：

```powershell
❯ echo $CEF_ROOT
/Users/w4ngzhen/projects/thirds/cef_binary_119.4.7+g55e15c8+chromium-119.0.6045.199_macosarm64
# 配置完成后，请确保环境变量生效
```

3）创建项目目录`cef_app_macos_project`，该目录将会存放本次macOS下工程的所有配置、源代码。

4）在项目根目录下创建`cmake`目录，并将步骤1中`cef_binary_xxx/cmake/FindCef.cmake`文件复制到`cmake`目录中：

![020-copy-FindCEF](https://static-res.zhen.wang/images/post/2023-12-12/020-copy-FindCEF.png)

## 项目根目录CMake配置

前期工作准备好以后，我们在项目根目录下创建`CMakeLists.txt`文件，并编写如下内容：

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.21)

PROJECT(cef_app_macos_project LANGUAGES CXX)

# 基础配置
SET(CMAKE_BUILD_TYPE DEBUG)
SET(CMAKE_CXX_STANDARD 17)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_INCLUDE_CURRENT_DIR ON)

# ===== CEF =====
if (NOT DEFINED ENV{CEF_ROOT})
    message(FATAL_ERROR "环境变量CEF_ROOT未定义！")
endif ()
# 执行下面之前，请确保环境变量CEF_ROOT已经配置为了对应cef_binary_xxx目录
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(CEF REQUIRED)

# ===== 子模块引入 =====
# 1. CEF前置准备完成后，此处便可以使用变量 CEF_LIBCEF_DLL_WRAPPER_PATH ，该值会返回libcef_dll_wrapper的目录地址
add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)
```

关于CMake本身的基础配置定义我们不再赘述，这里主要解释一下关于CEF引入的部分。首先，我们并没有把`cef_bin_xxx`目录复制到项目根目录下，而是放在了“外部”，并通过环境变量`CEF_ROOT`指向了它。在上述CMake关于CEF配置部分，我们对`CMAKE_MODULE_PATH`路径值追加了`cef_app_macos_project/cmake`目录。

> `${CMAKE_CURRENT_SOURCE_DIR}`就指代了项目根目录`cef_app_macos_project`。

接下来，在`find_package(CEF REQUIRED)`的时候，CMake会搜索`CMAKE_MODULE_PATH`路径下的名为`FindCEF.cmake`的CMake配置，于是就能找到我们曾复制的`cef_app_macos_project/cmake/FindCEF.cmake`文件并进行加载。

> 如果CMake初始化的时候出现了：
> 
> CMake Error at CMakeLists.txt:20 (message):
  环境变量CEF_ROOT未定义！
> 
> 请确保CEF_ROOT环境变量确定配置了。

对于`FindCEF.cmake`本身的内容，其核心逻辑就是读取环境变量`CEF_ROOT`值，然后定位到`cef_binary_xxx`目录，并加载`cef_binary_xxx/cmake/cef_variables.cmake`和`cef_binary_xxx/cmake/cef_macros.cmake`两个CMake配置文件。

> 这两个文件的作用分别是定义一些CEF提供的变量和宏方法，以便在后续的CMake加载逻辑中使用。

在`find_package`以后，我们调用了`add_subdirectory`指令，该指令第一个参数`${CEF_LIBCEF_DLL_WRAPPER_PATH}`就使用了来自`cef_variables.cmake`中定义值，指代了libcef_dll_wrapper代码工程的目录：

![030-libcef_dll_wrapper_var_path](https://static-res.zhen.wang/images/post/2023-12-12/030-libcef_dll_wrapper_var_path.png)

因此，这里的逻辑就是将`cef_binary_xxx/libcef_dll`目录作为了我们的CMake子模块工程，于是CMake会进一步加载`cef_binary_xxx/libcef_dll/CMakeLists.txt`文件并进行CMake相关文件的生成。细心的读者会注意到，这里还存在第二个参数`libcef_dll_wrapper`：

![040-pin-add_subdir_param](https://static-res.zhen.wang/images/post/2023-12-12/040-pin-add_subdir_param.png)

这里需要这个参数值的原因在于，`libcef_dll_wrapper`所在目录是一个**外部路径**，所以需要提供一个**目录名**作为的CMake文件二进制生成的路径。如果不提供，则会收到错误：

![050-add_subdir_no-param-error](https://static-res.zhen.wang/images/post/2023-12-12/050-add_subdir_no-param-error.png)

那么第二个参数具体影响了什么呢？如果读者使用CLion+CMake，会看到CLion会在项目根目录下生成`cmake-build-debug`目录，这个就是CMake生成文件目录，编译后的结果、CMake的过程文件都会在这个目录下找到（该目录其实就是cmake命令行的`-B`参数指定的路径，CLion默认指定的`项目根目录下/cmake-build-debug`目录）。在这里，当我们`add_subdirectory`添加了`libcef_dll_wrapper`子模块，经过CMake的初始化以后，会看到`cmake-build-debug/libcef_wrapper_dll`路径的产生：

![060-cmake-bin-dir-generate.png](https://static-res.zhen.wang/images/post/2023-12-12/060-cmake-bin-dir-generate.png)

至此，我们添加了对CEF的libcef_dll_wrapper子模块的引入，为了验证模块引入的正确性，我们尝试在当前`cef_app_macos_project`这个项目中对引入的子模块进行编译。有两种操作方式，方式1就是进入`cmake-build-debug`这个目录下使用命令：`cmake --build .`；当然，我们还可以使用IDE提供的更加便利的方式2：CLion直接使用GUI即可。

![070-build-dll-wrapper](https://static-res.zhen.wang/images/post/2023-12-12/070-build-dll-wrapper.png)

如果一切没有问题的情况下，我们可以在output目录中找到`libcef_dll_wrapper`的生成出来的库文件：

![080-libcef_dll_wrapper-build-ok](https://static-res.zhen.wang/images/post/2023-12-12/080-libcef_dll_wrapper-build-ok.png)

在继续后面的讲解前，我们先放慢脚步，对项目环境做一个总结。我们首先准备了两个目录，一个是我们自己的`cef_app_macos_project`目录，我们会在这个项目中“引入”CEF相关库，后续还会在里面编写我们自己的应用程序；另一个则是在外部的`cef_binary_xxx`目录，我们不会改动其中的内容。

对于我们自己的`cef_app_macos_project`，在根目录下，我们编写了一个`CMakeLists.txt`，它是我们项目顶层的CMake配置，该文件核心配置逻辑分以下几步：

1. 一些基本的项目、编译配置；
2. 加载CEF的CMake配置；
3. 引入外部的`cef_binary_xxx`中的`libcef_dll_wrapper`模块作为CMake子模块。

但请注意，目前我们仅仅是通过CMake提供的`add_subdirectory`命令，将`libcef_dll_wrapper`作为子模块引入，但目前还没有任何的应用在依赖它，接下来我们将进一步，开始配置主进程应用，并依赖该`libcef_dll_wrapper`。

## 主进程应用项目配置

在项目根目录下，我们创建`cef_app`目录，该目录目前先存放CEF的macOS应用的**主进程**应用项目代码。我们在`cef_app`目录下创建`process_main.mm`，且暂时先编写一段简单的代码：

```objc
#include <iostream>

int main(int argc, char *argv[]) {
  std::cout << "hello, this is main process." << std::endl;
  return 0;
}
```

> PS：`.mm`为后缀文件是指Objective-C与C/C++混写的源代码文件后缀，所以这里我们是可以完全写C++代码的。

然后，在`cef_app`目录中创建`CMakeLists.txt`文件，并编写如下的配置：

```cmake
# ===== 主进程target配置 =====
# 主进程target名称
set(CEF_APP_TARGET cef_app)
# 最终 App Bundle生成的路径
set(CEF_APP_BUNDLE "${CMAKE_CURRENT_BINARY_DIR}/${CEF_APP_TARGET}.app")
# 添加项目所有的源文件：
add_executable(
        ${CEF_APP_TARGET}
        MACOSX_BUNDLE # macOS 使用 "MACOSX_BUNDLE" 标识，最后编译产物是一个mac下的App Bundle
        process_main.mm
)
# 使用CEF提供的预定义好的工具宏，该宏会帮助配置target一些编译上的配置
# 如果出现不符合预期的编译结果、运行错误，可以检查该宏的内部实现
SET_EXECUTABLE_TARGET_PROPERTIES(${CEF_APP_TARGET})
# 添加对 libcef_dll_wrapper 库的依赖
# 基于该配置，可以保证每次编译当前 cef_app target时候，确保 libcef_dll_wrapper 静态库编译完成
add_dependencies(${CEF_APP_TARGET} libcef_dll_wrapper)
# 链接库配置
target_link_libraries(
        ${CEF_APP_TARGET}
        PRIVATE
        # libcef_dll_wrapper库链接
        libcef_dll_wrapper
        # 该变量来自cef_variables.cmake中定义的配置
        # 主要是针对不同的平台，链接对应平台的一些标准库（Windows、Linux）或者framework（macOS）
        ${CEF_STANDARD_LIBS}
)
# 主进程编译后，会在输出目录下生成一个名为 cef_app.app 的macOS App Bundle。
# 该app内部 Contents/MacOS/cef_app 仅仅是包含了 add_executable 中的源码二进制，以及libcef_dll_wrapper静态库
# 在macOS下，我们还需要将"cef_binary_xxx/Debug或Release目录/Chromium Embedded Framework.framework"复制到
# cef_app.app/Contents/Frameworks目录下
# 为了避免手动复制的麻烦，我们使用如下的指令完成复制工作
add_custom_command(
        # 对 CEF_APP_TARGET 进行操作
        TARGET ${CEF_APP_TARGET}
        # 在构建完成后（POST_BUILD）
        POST_BUILD
        # COMMAND ${CMAKE_COMMAND}：就是命令行执行 "cmake"
        # -E：指可以执行一些cmake内置的工具命令
        # copy_directory：进行目录复制操作
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        # 复制源目录、文件，
        # CEF_BINARY_DIR变量来源于cef_variables.cmake
        # 等价于"cef_binary_xxx目录/Debug或Release目录/"
        "${CEF_BINARY_DIR}/Chromium Embedded Framework.framework"
        # 将上述 framework 复制到 当前生成的 cef_app.app/Contents/Frameworks/对应framework名称
        "${CEF_APP_BUNDLE}/Contents/Frameworks/Chromium Embedded Framework.framework"
        # 不进行文本的解析，使用源文字，考虑会有表达式情况
        VERBATIM
)
# 简单配置Info.plist的一些值
set_target_properties(
        ${CEF_APP_TARGET}
        PROPERTIES
        MACOSX_BUNDLE_BUNDLE_NAME ${CEF_APP_TARGET}
        MACOSX_BUNDLE_GUI_IDENTIFIER ${CEF_APP_TARGET}
)
```

我们接下来对上述的配置逐一解释：

```cmake
# 主进程target名称
set(CEF_APP_TARGET cef_app)
# 最终 App Bundle生成的路径
set(CEF_APP_BUNDLE "${CMAKE_CURRENT_BINARY_DIR}/${CEF_APP_TARGET}.app")
```

上述配置了我们接下来将会定义的target的名称，以及后续生成的macOS特有的App Bundle的应用文件的路径，后续会使用到该值。

```cmake
add_executable(
        ${CEF_APP_TARGET}
        MACOSX_BUNDLE # macOS 使用 "MACOSX_BUNDLE" 标识，最后编译产物是一个mac下的App Bundle
        process_main.mm
)
```

`add_executable`部分定义最终生成的target，除了包含编写的源码路径（`process_main.mm`），这里还有一个很重要的参数`MACOS_BUNDLE`，配置该参数后，在macOS下，我们最终生成的可执行程序就不再是一个简单的命令行程序，而是macOS下的App Bundle。下图是没有配置该值前后的对比：

![090-MACOS_BUNDLE-param-diff](https://static-res.zhen.wang/images/post/2023-12-12/090-MACOS_BUNDLE-param-diff.png)

可以看到，没有配置`MACOSX_BUNDLE`时，最终项目会在输出目录（`${CMAKE_CURRENT_BINARY_DIR}`）下生成名为`cef_app`的可执行命令行程序；而配置以后，项目会在输出目录下生成`target名.app`，这里就是`cef_app.app`。

```cmake
# 使用CEF提供的预定义好的工具宏，该宏会帮助配置target一些编译上的配置
# 如果出现不符合预期的编译结果、运行错误，可以检查该宏的内部实现
SET_EXECUTABLE_TARGET_PROPERTIES(${CEF_APP_TARGET})
```

`SET_EXECUTABLE_TARGET_PROPERTIES`**不是CMake提供的指令，而是由CEF提供的**，存放于`cef_macros.cmake`中的宏。该宏主要的功能是对目标target配置一些可执行程序所需要的编译参数等。如果读者在实践过程中，遇到了链接问题，可以优先检查这个宏中的实现。由于篇幅原因，这块后续单独出一篇文章水一水，>_<。

```cmake
# 添加对 libcef_dll_wrapper 库的依赖
# 基于该配置，可以保证每次编译当前 cef_app target时候，确保 libcef_dll_wrapper 静态库编译完成
add_dependencies(${CEF_APP_TARGET} libcef_dll_wrapper)
```

`add_dependencies`的作用则是为当前target指定依赖。因为我们的项目本身会通过静态链接库的形式链接`libcef_dll_wrapper`，通过这`add_dependencies`能够保证最终构建过程中，确保优先将`libcef_dll_wrapper`编译出来，供后续链接过程使用。当然，你也可以不闲麻烦的手动先编译`libcef_dll_wrapper`，再编译这个`cef_app`。

```cmake
# 链接库配置
target_link_libraries(
        ${CEF_APP_TARGET}
        PRIVATE
        # libcef_dll_wrapper库链接
        libcef_dll_wrapper
        # 该变量来自cef_variables.cmake中定义的配置
        # 主要是针对不同的平台，链接对应平台的一些标准库（Windows、Linux）或者framework（macOS）
        ${CEF_STANDARD_LIBS}
)
```

`target_link_libraries`处理则是配置当前target的链接库，包括不限于libcef_dll_wrapper的静态链接、各种平台特定的链接库等。最后一个参数变量`CEF_STANDARD_LIBS`，由CEF在`cef_variables.cmake`中定义，包含平台特定的链接库。

> 例如，在Windows下我们可能需要`gdi32.lib`，在Linux构建窗体可能需要X11库，以及在macOS下需要`Cocoa`、`AppKit`等框架库。读者可以翻阅`cef_variables.cmake`中关于这个变量的配置了解具体的内容。

```cmake
# 主进程编译后，会在输出目录下生成一个名为 cef_app.app 的macOS App Bundle。
# 该app内部 Contents/MacOS/cef_app 仅仅是包含了 add_executable 中的源码二进制，以及libcef_dll_wrapper静态库
# 在macOS下，我们还需要将"cef_binary_xxx/Debug或Release目录/Chromium Embedded Framework.framework"复制到
# cef_app.app/Contents/Frameworks目录下
# 为了避免手动复制的麻烦，我们使用如下的指令完成复制工作
add_custom_command(
        # 对 CEF_APP_TARGET 进行操作
        TARGET ${CEF_APP_TARGET}
        # 在构建完成后（POST_BUILD）
        POST_BUILD
        # COMMAND ${CMAKE_COMMAND}：就是命令行执行 "cmake"
        # -E：指可以执行一些cmake内置的工具命令
        # copy_directory：进行目录复制操作
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        # 复制源目录、文件，
        # CEF_BINARY_DIR变量来源于cef_variables.cmake
        # 等价于"cef_binary_xxx目录/Debug或Release目录/"
        "${CEF_BINARY_DIR}/Chromium Embedded Framework.framework"
        # 将上述 framework 复制到 当前生成的 cef_app.app/Contents/Frameworks/对应framework名称
        "${CEF_APP_BUNDLE}/Contents/Frameworks/Chromium Embedded Framework.framework"
        # 不进行文本的解析，使用源文字，考虑会有表达式情况
        VERBATIM
)
```

倒数第二个指令`add_custom_command`，在介绍它的作用前，先简单说明在macOS下基于CEF的App Bundle的一应用结构。基于前面的配置，主进程编译后，会在输出目录下生成一个名为`cef_app.app`的macOS App Bundle，该Bundle内部`/Contents/MacOS/cef_app`可执行程序，就是链接了源码二进制、libcef_dll_wrapper静态库后的可执行二进制程序。然而，CEF核心库`Chromium Embedded Framework.framework`我们并没有静态链接到执行程序内，而是在实际运行过程中，动态加载这个framework。为了达到该目的，我们思路是通过脚本将`cef_binary_xxx`中提供的CEF的核心库framework拷贝到App Bundle中指定路径下。

所以，在了解了App Bundle运行逻辑以后，关于`add_custom_command`作用就显而易见了，其逻辑就是配置在构建完成以后，通过CMake的工具指令（`-E copy_directories`）将`Chromium Embedded Framework.framework`整个内容复制到生成的Bundle的`/Contents/Frameworks`目录下：

![100-copy-CEF-framework](https://static-res.zhen.wang/images/post/2023-12-12/100-copy-CEF-framework.png)

在上面的讲解中我们大致理解了macOS的App Bundle的应用程序组织结构，细心的读者会发现，在构建后的Bundle中的根目录下有一个文件`Info.plist`：

![110-info-plist-file](https://static-res.zhen.wang/images/post/2023-12-12/110-info-plist-file.png)

该文件的核心作用是定义macOS下App Bundle的基础应用程序配置，包括不限于该应用的名称、应用ID、图标资源等。因为我们将主进程target定义为了`MACOS_BUNDLE`，CMake会在构建的时候，默认为我们的Bundle生成了一份plist并写入到Bundle中。同时我们会发现，`Info.plist`配置中关于`CFBundleName`、`CFBundleIdentifier`等值就是我们现在的target的名称：

![120-info-plist-content](https://static-res.zhen.wang/images/post/2023-12-12/120-info-plist-content.png)

原因在于配置文件中紧接着`add_custom_command`后面的`set_target_properties`：

```cmake
# 简单配置Info.plist的一些值
set_target_properties(
        ${CEF_APP_TARGET}
        PROPERTIES
        MACOSX_BUNDLE_BUNDLE_NAME ${CEF_APP_TARGET}
        MACOSX_BUNDLE_GUI_IDENTIFIER ${CEF_APP_TARGET}
)
```

使用`set_target_properties`指令指定了`MACOSX_BUNDLE_BUNDLE_NAME`和`MACOSX_BUNDLE_GUI_IDENTIFIER`的值。关于这段配置的说明，官方文档提到：https://cmake.org/cmake/help/latest/prop_tgt/MACOSX_BUNDLE_INFO_PLIST.html，我们可以直接通过相关属性值来替换CMake内置的plist模板文件内容。

> 注意，CMake支持的变量只有上述官方文档提供的Key，如果有其他的Key需要处理，只能通过自己提供模板方法进行处理，这点会在后面构建子进程Bundle再次说明。

至此，我们基本完成了在macOS对主进程的CMake配置。此时，请务必注意，记得在项目根目录的CMakeLists.txt追加如下将`cef_app`目录作为子模块引入的配置：

```diff
# 1. CEF前置准备完成后，此处便可以使用变量 CEF_LIBCEF_DLL_WRAPPER_PATH ，该值会返回libcef_dll_wrapper的目录地址
add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)
+ # 2. 将cef_app作为子模块引入
+ add_subdirectory(./cef_app)
```

当然，我们主进程应用的源代码还是只是简单的在控制台输出一段话，我们不着急编写主进程代码，接下来还需要配置对应的子进程项目。

## 子进程应用项目配置

我们在一开始已经提到过，在macOS建议将主进程和子进程分别构建为两个不同的App Bundle，这里我们有两种做法：

- 方式1：通过CMake的定义target，在前面主进程CMakeLists.txt中直接定义子进程的target，让构建系统同时生成另外的子进程应用。

- 方式2：直接重新创建一个目录来定义子进程CMake模块并存放子进程模块代码。

这里笔者使用第一种方式来进行配置，或许配置上略显复杂，但只要读者一旦理解，笔者相信今后对于其他CMake项目配置应该也能很快上手。

我们先在`cef_app`目录中创建一个名为`process_helper.mm`的文件，暂时作为子进程的入口源码：

```objc
#include <iostream>

int main(int argc, char *argv[]) {
  std::cout << "hello, this is sub helper process." << std::endl;
  return 0;
}
```

同时，在该子模块目录下创建一个`templates`目录，并在其中创建`helper-Info.plist`文件，具体的意义和其内容我们后面介绍，这里读者可以将它理解为一份模板文件。

此时，我们的项目结构如下：

![130-sub-process-new-file](https://static-res.zhen.wang/images/post/2023-12-12/130-sub-process-new-file.png)

> 为了阅读的方便，我们都将子进程叫做helper

接下来，我们在`cef_app/CMakeLists.txt`内容的基础上，添加如下的针对helper子进程应用的配置：

```cmake
# ===== 主进程target配置 =====
# ... ...
# ===== 子进程 helper target配置 =====
# 定义helper子进程target名
set(CEF_APP_HELPER_TARGET "cef_app_helper")
# 定义helper子进程构建后的app的名称
set(CEF_APP_HELPER_OUTPUT_NAME "cef_app Helper")
# 注意，上述的名称都不是最终名称，它们更准确的意义是作为下面循环定义target的基础名称
# 后续循环的时候，会基于上述名称进行拼接

# 创建多个不同类型helper的target
# CEF_HELPER_APP_SUFFIXES来自cef_variables.cmake，是一个“字符串数组”，值有：
# "::"、" (Alerts):_alerts:.alerts"、" (GPU):_gpu:.gpu"、
# " (Plugin):_plugin:.plugin"、" (Renderer):_renderer:.renderer"
# 这里通过foreach，实现对字符串数组的遍历，每一次循环会得到一个字符串，存放在“_suffix_list”
foreach (_suffix_list ${CEF_HELPER_APP_SUFFIXES})
  # 将字符串转为";"分割，这样可以使用CMake支持的list(GET)指令来读取每一节字符串
  # 以 " (Renderer):_renderer:.renderer" 为例
  string(REPLACE ":" ";" _suffix_list ${_suffix_list}) # " (Renderer);_renderer;.renderer"
  list(GET _suffix_list 0 _name_suffix) # " (Renderer)"
  list(GET _suffix_list 1 _target_suffix) # "_renderer"
  list(GET _suffix_list 2 _plist_suffix) # ".renderer"
  # 当然，需要注意 CEF_HELPER_APP_SUFFIXES 中有一个"::"的字符串，
  # 会使得 _name_suffix = ""、_target_suffix = ""、_plist_suffix = ""

  # 定义一个Helper target以及BUNDLE名称
  # 以 " (Renderer):_renderer:.renderer" 为例
  # _helper_target = "cef_app_helper" + "_renderer" -> "cef_app_helper_renderer"
  # _helper_output_name = "cef_app Helper" + " (Renderer)" -> "cef_app Helper (Renderer)"
  set(_helper_target "${CEF_APP_HELPER_TARGET}${_target_suffix}")
  set(_helper_output_name "${CEF_APP_HELPER_OUTPUT_NAME}${_name_suffix}")

  # 读取templates/helper-Info.plist模板文件内容到_plist_contents
  # 然后使用上面得到的 _helper_output_name、_plist_suffix等变量进行文本内容的替换操作
  # 以便得到当前正在处理的helper对应的一份Info.plist
  file(READ "${CMAKE_CURRENT_SOURCE_DIR}/templates/helper-Info.plist" _plist_contents)
  string(REPLACE "\${HELPER_EXECUTABLE_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
  string(REPLACE "\${PRODUCT_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
  string(REPLACE "\${BUNDLE_ID_SUFFIX}" "${_plist_suffix}" _plist_contents ${_plist_contents})
  # helper的Info.plist文件路径，例如："${CMAKE_CURRENT_BINARY_DIR}/helper-Info[_renderer].plist"
  set(_helper_info_plist_file "${CMAKE_CURRENT_BINARY_DIR}/helper-Info${_target_suffix}.plist")
  # 通过CMake提供file(WRITE)命令，将前面定义的内容写入到对应.plist文件中
  file(WRITE ${_helper_info_plist_file} ${_plist_contents})

  # 创建当前helper的executable target，当然，也是一个App Bundle
  add_executable(${_helper_target}
      MACOSX_BUNDLE
      process_helper.mm
  )
  # 与主进程应用一样，
  # 通过cef提供的SET_EXECUTABLE_TARGET_PROPERTIES宏，来设置编译参数、头文件路径等
  SET_EXECUTABLE_TARGET_PROPERTIES(${_helper_target})
  # 编译当前Helper target前，先编译 libcef_dll_wrapper target
  add_dependencies(${_helper_target} libcef_dll_wrapper)
  # 当前Helper target的库链接
  target_link_libraries(${_helper_target} libcef_dll_wrapper ${CEF_STANDARD_LIBS})
  # 定义当前Helper target的一些属性
  set_target_properties(${_helper_target} PROPERTIES
      # 这里使用“MACOSX_BUNDLE_INFO_PLIST”，
      # 来定义构建过程Bundle使用的Info.plist来源于前面我们通过模板文件生成的.plist
      MACOSX_BUNDLE_INFO_PLIST ${_helper_info_plist_file}
      # 定义最终生成的App Bundle的名称
      OUTPUT_NAME ${_helper_output_name}
  )

  # 构建主进程应用前，会先构建当前Helper target
  add_dependencies(${CEF_APP_TARGET} "${_helper_target}")

  # 将构建的Helper App Bundle拷贝到主进程cef_app的Bundle中
  add_custom_command(
      TARGET ${CEF_APP_TARGET}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
      "${CMAKE_CURRENT_BINARY_DIR}/${_helper_output_name}.app"
      "${CEF_APP_BUNDLE}/Contents/Frameworks/${_helper_output_name}.app"
      VERBATIM
  )
endforeach ()
```

让我们从头到尾一一道来。

```cmake
# 定义helper子进程target名
set(CEF_APP_HELPER_TARGET "cef_app_helper")
# 定义helper子进程构建后的app的名称
set(CEF_APP_HELPER_OUTPUT_NAME "cef_app Helper")
# 注意，上述的名称都不是最终名称，它们更准确的意义是作为下面循环定义target的基础名称
# 后续循环的时候，会基于上述名称进行拼接
```

首先，我们会定义helper子进程的target名称和输出应用名称。但需要注意的是，这里的名称不完全是最终输出的应用程序的名称。因为在后续的配置中，我们会使用CMake支持的循环命令来支持生成多个target。

```cmake
# 创建多个不同类型helper的target
# CEF_HELPER_APP_SUFFIXES来自cef_variables.cmake，是一个“字符串数组”，值有：
# "::"、" (Alerts):_alerts:.alerts"、" (GPU):_gpu:.gpu"、
# " (Plugin):_plugin:.plugin"、" (Renderer):_renderer:.renderer"
# 这里通过foreach，实现对字符串数组的遍历，每一次循环会得到一个字符串，存放在“_suffix_list”
foreach (_suffix_list ${CEF_HELPER_APP_SUFFIXES})
 ... ...
endforeach ()
```

接着，我们使用CMake的`foreach`指令，来遍历变量`CEF_HELPER_APP_SUFFIXES`这个变量值。这个变量来自于cef提供的变量（cef_variables.cmake）：

```cmake
  # CEF Helper app suffixes.
  # Format is "<name suffix>:<target suffix>:<plist suffix>".
  set(CEF_HELPER_APP_SUFFIXES
    "::"
    " (Alerts):_alerts:.alerts"
    " (GPU):_gpu:.gpu"
    " (Plugin):_plugin:.plugin"
    " (Renderer):_renderer:.renderer"
    )
```

在这里通过CMake的遍历能力，我们每一次迭代都能读取到对应一条字符串并存放到`_suffix_list`变量中。

接下来介绍在`foreach`包裹的内部配置：

```cmake
    # 将字符串转为";"分割，这样可以使用CMake支持的list(GET)指令来读取每一节字符串
    # 以 " (Renderer):_renderer:.renderer" 为例
    string(REPLACE ":" ";" _suffix_list ${_suffix_list}) # " (Renderer);_renderer;.renderer"
    list(GET _suffix_list 0 _name_suffix) # " (Renderer)"
    list(GET _suffix_list 1 _target_suffix) # "_renderer"
    list(GET _suffix_list 2 _plist_suffix) # ".renderer"
    # 当然，需要注意 CEF_HELPER_APP_SUFFIXES 中有一个"::"的字符串，
    # 会使得 _name_suffix = ""、_target_suffix = ""、_plist_suffix = ""
```

我们将`_suffix_list`变量中所有的`:`字符替换为`;`，然后就可以使用CMake支持的`list(GET)`指令来读取每一节字符串。

以 `" (Renderer):_renderer:.renderer"`为例，在替换后，通过`list(GET)`可以分别得到：

- _name_suffix = `" (Renderer)"`
- _target_suffix = `"_renderer"`
- _plist_suffix = `".renderer"`

这三个suffix将在后续的流程拼接出相关名称变量。但需要注意的是，在`CEF_HELPER_APP_SUFFIXES`中存在一个特殊的字符串：`"::"`。这个字符串会导致最后提取出来的前面三个suffix都是`""`（空字符串），这并不是BUG，后续会用到。

```cmake
    # 定义一个Helper target以及BUNDLE名称
    # 以 " (Renderer):_renderer:.renderer" 为例
    # _helper_target = "cef_app_helper" + "_renderer" -> "cef_app_helper_renderer"
    # _helper_output_name = "cef_app Helper" + " (Renderer)" -> "cef_app Helper (Renderer)"
    set(_helper_target "${CEF_APP_HELPER_TARGET}${_target_suffix}")
    set(_helper_output_name "${CEF_APP_HELPER_OUTPUT_NAME}${_name_suffix}")
```

接下来，我们开始消费suffix。首先，我们通过拼接操作得到`_helper_target`和`_helper_output_name`。这两个变量分别代表了当前正在构建的helper的真正target名和对应后续构建的应用名称。还是以 `" (Renderer):_renderer:.renderer"`为例。我们能够得到：

- `_helper_target` = `"cef_app_helper" + "_renderer"` 得到 `"cef_app_helper_renderer"`
- `_helper_output_name` = `"cef_app Helper" + " (Renderer)"` 得到 `"cef_app Helper (Renderer)"`

```cmake
    # 读取templates/helper-Info.plist模板文件内容到_plist_contents
    # 然后使用上面得到的 _helper_output_name、_plist_suffix等变量进行文本内容的替换操作
    # 以便得到当前正在处理的helper对应的一份Info.plist
    file(READ "${CMAKE_CURRENT_SOURCE_DIR}/templates/helper-Info.plist" _plist_contents)
    string(REPLACE "\${HELPER_EXECUTABLE_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
    string(REPLACE "\${PRODUCT_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
    string(REPLACE "\${BUNDLE_ID_SUFFIX}" "${_plist_suffix}" _plist_contents ${_plist_contents})
    # helper的Info.plist文件路径，例如："${CMAKE_CURRENT_BINARY_DIR}/helper-Info[_renderer].plist"
    set(_helper_info_plist_file "${CMAKE_CURRENT_BINARY_DIR}/helper-Info${_target_suffix}.plist")
    # 通过CMake提供file(WRITE)命令，将前面定义的内容写入到对应.plist文件中
    file(WRITE ${_helper_info_plist_file} ${_plist_contents})
```

接下来，我们使用CMake提供的能力，读取了前面提到的存放在`cef_app/templates`目录下的`helper-Info.plist`文件。这是一个模板文件，打开后读者能从中看到一些`${XXX}`的占位字符串，我们会在这一步进行对应文本的替换。这里我们用到了CMake的几个知识点：

1. file(READ)读取某个文件并存放到文本变量中；
2. string(REPLAECE)替换文本变量中某些字符串并写回到变量中；
3. file(WRITE)将文本数据写入到某个文件中。

这一步我们还得到了`_helper_info_plist_file`变量，它指向了我们写入的plist文件，以便在后续配置中进行使用。

```cmake
    # 创建当前helper的executable target，当然，也是一个App Bundle
    add_executable(${_helper_target}
            MACOSX_BUNDLE
            process_helper.mm
    )
    # 与主进程应用一样，
    # 通过cef提供的SET_EXECUTABLE_TARGET_PROPERTIES宏，来设置编译参数、头文件路径等
    SET_EXECUTABLE_TARGET_PROPERTIES(${_helper_target})
    # 编译当前Helper target前，先编译 libcef_dll_wrapper target
    add_dependencies(${_helper_target} libcef_dll_wrapper)
    # 当前Helper target的库链接
    target_link_libraries(${_helper_target} libcef_dll_wrapper ${CEF_STANDARD_LIBS})
    # 定义当前Helper target的一些属性
    set_target_properties(${_helper_target} PROPERTIES
            # 这里使用“MACOSX_BUNDLE_INFO_PLIST”，
            # 来定义构建过程Bundle使用的Info.plist来源于前面我们通过模板文件生成的.plist
            MACOSX_BUNDLE_INFO_PLIST ${_helper_info_plist_file}
            # 定义最终生成的App Bundle的名称
            OUTPUT_NAME ${_helper_output_name}
    )
```

和前面主进程应用target类似。我们将helper的构建结果同样定义为App Bundle；使用`SET_EXECUTABLE_TARGET_PROPERTIES`来进行编译参数等设置；使用`add_dependencies`告诉CMake编译构建子进程target的时候，保证libcef_dll_wrapper优先于helper构建完成；使用`target_link_libraries`链接子进程Helper。但，最后一个`set_target_properties`和之前主进程target设置有所不同。在之前的主进程应用配置时，我们直接使用了诸如`MACOSX_BUNDLE_BUNDLE_NAME`、`MACOSX_BUNDLE_GUI_IDENTIFIER`等参数来让CMake使用内置的plist模板文件生成主进程应用App Bundle中的plist文件。但因为CMake内置的模板plist只能设置部分字段值，而在Helper配置的时候，我们需要更改更多的占位字段，所以我们自己提供了helper Bundle的模板plist，并通过内容读取、字符串替换的方式生成了对应Helper的Bundle的plist文件内容。要让CMake不再使用内置的模板plist，而是使用我们生成的plist文件，我们使用参数`MACOSX_BUNDLE_INFO_PLIST`指定前面生成好的plist文件路径。最后，我们还定义了`OUTPUT_NAME`这个参数，这个参数主要的作用是可以自定义生成的应用程序的名称，如果没有这个参数，我们最终在构建结果目录中生成应用名称就是target。

```cmake
    # 构建主进程应用前，会先构建当前Helper target
    add_dependencies(${CEF_APP_TARGET} "${_helper_target}")
```

告诉CMake，构建主进程target应用的时候，会先构建当前Helper target。

```cmake
    # 将构建的Helper App Bundle拷贝到主进程cef_app的Bundle中
    add_custom_command(
            TARGET ${CEF_APP_TARGET}
            POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${CMAKE_CURRENT_BINARY_DIR}/${_helper_output_name}.app"
            "${CEF_APP_BUNDLE}/Contents/Frameworks/${_helper_output_name}.app"
            VERBATIM
    )
```

在循环的最后，我们再次使用`add_custom_command`通过CMake提供的文件复制能力，让主进程应用构建完成以后，将当前子进程helper应用app复制到`主进程应用.app/Contents/Frameworks`目录下。至于为什么要这么做，我们将会在下一篇文章中介绍应用程序运行时架构来说明。

基于现在完成的配置，我们可以通过对cef_app进行构建，检查最终构建的产物来验证项目的正确性。笔者使用CLion的GUI生成cef_app，最终会在输出目录中找到cef_app.app，同时会看到会生成多个helper的App Bundle，并已经成功复制到了对应目录中：

![140-build-result](https://static-res.zhen.wang/images/post/2023-12-12/140-build-result.png)

# 写在最后

在本文，我们基本上完成了在macOS下基于CEF的多进程应用架构的项目CMake配置，并结合实际的配置，逐一说明了CMake的相关用法和配置细节。在下一篇文章中，我们会基于此文搭建的项目，逐步介绍并编写macOS下基于CEF应用程序的代码，其中会涉及到macOS下Cocoa框架知识简介。

本文仓库链接：[w4ngzhen/cef_app_macos_project (github.com)](https://github.com/w4ngzhen/cef_app_macos_project)
