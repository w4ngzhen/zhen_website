---
title: IDEA Web渲染插件开发（一）— 使用JCEF
date: 2021-07-16
tags:
 - jcef
 - idea plugin
categories: 
- 技术
- IDEA Web渲染插件开发
---

目前网上已经有了很多关于IDEA（IntelliJ平台）的插件开发教程了，本人觉得简书上这位作者[秋水畏寒 ](https://www.jianshu.com/u/3adf83c26b4e)的关于插件开发的文章很不错，在我进行插件开发的过程中指导了我很多。但是综合下来看，在IDEA上加载网页的插件的教程还不是特别多，官方文档也不是那么的完整。本系列将会从这个角度出发，探讨如何编写加载Web页面的插件。

<!-- more -->

# 前言

为什么会有想到开发处理Web网页的插件呢？实际上因为在IDEA中，我们可以打开markdown文件，并且IDEA具有markdown实时渲染的能力：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/010-idea-md-show.jpg)

因为之前，本人使用过JCEF进行开发。看到这个渲染，心里大概猜测，应该用了浏览器内核。打开任务管理器：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/020-idea-use-jcef.jpg)

果然，熟悉的JCEF。然后进入JetBrains的官网，在插件开发的文档中找到了：[JCEF - Java Chromium Embedded Framework | IntelliJ Platform Plugin SDK (jetbrains.com)](https://plugins.jetbrains.com/docs/intellij/jcef.html)。

那么，接下来我们从零开始，编写一款属于自己的插件，这款插件能够加载Web页面。

# 环境准备

- JDK 11
- Gradle
- 良好的网络环境

我们先创建一个IntelliJ Platform Plugin，名为：intellij-jcef-plugin

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/030-create-plugin-proj-1.jpg)

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/040-create-plugin-proj-2.jpg)

然后进行这个Gradle项目的配置工作，完成整个项目搭建。本项目会在最后提交到github供读者下载。

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/050-proj-arch.jpg)

# 代码编写

首先说明我们的目的，就是希望能够类似于gradle、maven插件一样，能够在IDEA的侧边有一个显示我们Web页面的地方：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/060-display-like-gradle-page.jpg)

通过阅读官方的文档我们可以知道，我们需要编写的是ToolWindow（[Tool Windows | IntelliJ Platform Plugin SDK (jetbrains.com)](https://plugins.jetbrains.com/docs/intellij/tool-windows.html)）这样一个页面窗体。

## 基础ToolWindow开发

在开发之前，我们需要明确一点，尽管这一节的标题写着"空白ToolWindow开发"，似乎在暗示我们，接下来我们会开发一个所谓的ToolWindow的实现类。实际上，ToolWindow是插件框架本身提供的，我们只需要做的是创建UI组件（例如JPanel），然后调用ToolWindow实例通过相关的API帮我们把UI组件设置到ToolWindow内部，具体的步骤如下：

### 实现ToolWindowFactory

创建一个ToolWindowFactory的实现类，这里我们取名MyToolWindowFactory，然后重写`createToolWindowContent`方法。

```java
public class MyToolWindowFactory implements ToolWindowFactory {
    @Override
    public void createToolWindowContent(
            @NotNull Project project,
            @NotNull ToolWindow toolWindow) {
        // 此处方法将会在点击ToolWindow的时候触发
        // 获取ContentManager
        ContentManager contentManager = toolWindow.getContentManager();
        Content labelContent =
                contentManager.getFactory() // 内容管理器获取工厂类
                        .createContent( // 创建Content（组件类实例、显示名称、是否可以锁定）
                                new JLabel("hello, world"),
                                "MyTab",
                                false
                        );
        // 利用ContentManager添加Content
        contentManager.addContent(labelContent);
    }
}
```

在重写的`createToolWindowContent`方法中，插件框架会为我们传入两个对象：Project以及ToolWindow对象。其中，Project对象是当前项目的内容抽象，而ToolWindow这个对象就是插件框架本身内部构造的，抽象了我们需求所说的，点击侧边栏时候弹出的页面。

在该方法实现中，主要有以下步骤：

1. 使用ContentFactory（ContentManager.getFactory()获取）的`createContent`API创建Content对象。这个创建时候，需要swing组件对象（JPanel、JLabel等等）。
2. 使用ContentManager的`addContent`API添加步骤1的Content对象。

### 注册插件

接下来，我们将我们实现的MyToolWindowFactory通过`plugin.xml`进行注册，`alt+enter`，IDEA帮助我们快速完成填写xml配置到`plugin.xml`中：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/070-register-toolwindow.jpg)

进行上述操作后，IDEA自动为我们在plugin.xml文件的`extensions节点`中，添加了`toolWindow节点`的内容，但是我们还需要填写必备的属性`id`：

```xml
<!-- plugin.xml文件 -->
    <extensions defaultExtensionNs="com.intellij">
        <!-- Add your extensions here -->
        <!-- id是必须的属性，我们进行添加 -->
        <!-- anchor锚点非必须，但是为了像Gradle插件一样默认显示在右边，我们设置为right -->
        <toolWindow id="myToolWindowFactory"
                    anchor="right"
                    factoryClass="com.compilemind.demo.ui.MyToolWindowFactory"
        />
    </extensions>
```

### 解决调试环境问题

目前为止，我们实现了ToolWindowFactory以及将我们的实现类注册到plugin.xml中。现在，我们先什么内容都不编写，开始调试我们的插件：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/080-begin-debug.jpg)

不过开始调试后，会有很多的情况发生，这里我做了一些遇到的问题的总结。

#### Gradle乱码

此时进行Debug调试，在我的机器上会出现乱码：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/090-debug-but-encode-error.jpg)

解决方案为，在build.gradle中添加如下的语句：

```groovy
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}
```

#### Gradle报错不知道这样的主机（Unknown host）

如果出现了类似于`Unknown host 'xxxxx.cloudfront.net'. You may need to adjust the proxy settings in Gradle.`这样的报错，一般是当前网络的连通问题，导致无法下载cloudfront.net一些jar文件。此时挂代理是最好的办法。

#### rumIde：Download JCEF

如果使用调试模式，intellij插件开发的Gradle插件会下载jcef的运行时，这个过程会比较漫长，目前解决办法是使用好的网络等待下载：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/100-downloading-jcef.jpg)

在本人机器上，第一次调试的时候主要就是遇到上面的三种情况。

### 验证基础ToolWindow

解决完上述的几个问题之后，界面弹出了我们的调试下的社区版的IDEA（ideaIC），并且，查看Plugins页签，会发现我们编写的插件已经被这个ideaIC安装了：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/110-first-run-ideaIC-with-my-plugin.jpg)

我们使用这个IDEA创建一个简单的空项目，然后可以看到右侧有我们提供的ToolWindow：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/120-first-display-my-toolwindow.jpg)

可以看到，此时的ToolWindow中的内容显示为我们上面设置的`new JLabel("hello, world")`，该ToolWindow上方有我们设置的"My Tab"标题。截至目前的代码，包含在这个github上这个提交：

[simple ToolWindow Content · w4ngzhen/intellij-jcef-plugin@bf2ca8e (github.com)](https://github.com/w4ngzhen/intellij-jcef-plugin/commit/bf2ca8eb71c36a46077d222b031439537d8015cd)

## Web页面ToolWindow开发

通过上面一些系列的环境搭建，以及ToolWindow开发练习，我们已经了解了如何开发一款用于IDEA侧边栏展示内容的插件。当然，我们一开始的需求是要在ToolWindow中展示网页，并且也知道了，JetBrains已经将JCEF引入到了IntelliJ插件平台。接下来，我们使用JCef以及JBCef相关API创建一个用于展示Web的UI组件，再通过上述的方式，添加到ToolWindow。

### 创建MyWebToolWindowContent

```java
package com.compilemind.demo.ui;

import com.intellij.ui.jcef.JBCefApp;
import com.intellij.ui.jcef.JBCefBrowser;

import javax.swing.*;
import java.awt.*;

public class MyWebToolWindowContent {

    private final JPanel content;

    /**
     * 构造函数
     */
    public MyWebToolWindowContent() {
        this.content = new JPanel(new BorderLayout());
        // 判断所处的IDEA环境是否支持JCEF
        if (!JBCefApp.isSupported()) {
            this.content.add(new JLabel("当前环境不支持JCEF", SwingConstants.CENTER));
            return;
        }
        // 创建 JBCefBrowser
        JBCefBrowser jbCefBrowser = new JBCefBrowser();
        // 将 JBCefBrowser 的UI控件设置到Panel中
        this.content.add(jbCefBrowser.getComponent(), BorderLayout.CENTER);
        // 加载URL
        jbCefBrowser.loadURL("https://cnblogs.com/w4ngzhen");
    }

    /**
     * 返回创建的JPanel
     * @return JPanel
     */
    public JPanel getContent() {
        return content;
    }
}
```

### 修改MyToolWindowFactory

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/130-modify-factory.jpg)

这里，我们将创建`MyWebToolWindowContent`实例，然后返回其Panel，按同样的方式设置到ToolWindow中。

### 验证Web渲染ToolWindow

上述代码完成开发后，我们再次运行Debug模式，可以看到此时的界面显示了相关的网页：

![](https://res.zhen.wang/images/post/2021-07-16-intelliJ-plugin-dev-1/140-display-toolwindow-with-web-page.jpg)

# 附录

本次代码本人放在了Github上，地址为：[w4ngzhen/intellij-jcef-plugin (github.com)](https://github.com/w4ngzhen/intellij-jcef-plugin)。

上面**基础ToolWindow开发**以及**web页面ToolWindow开发**两节的内容，按如下提交对应：

基础ToolWindow开发 ：[simple ToolWindow Content · w4ngzhen/intellij-jcef-plugin@bf2ca8e (github.com)](https://github.com/w4ngzhen/intellij-jcef-plugin/commit/bf2ca8eb71c36a46077d222b031439537d8015cd)

web页面ToolWindow开发：[web ToolWindow Content · w4ngzhen/intellij-jcef-plugin@45604d3 (github.com)](https://github.com/w4ngzhen/intellij-jcef-plugin/commit/45604d374eaead417b16df2ade1b5d6700e291f3)

