---
title: IDEA Web渲染插件开发（二）— 自定义JsDialog
date: 2021-10-06
tags: 
 - IDEA
 - Plugins 
categories:
  - 技术
  - IDEA Web渲染插件开发
---

《IDEA Web渲染插件开发（一）》中，我们了解到了如何编写一款用于显示网页的插件，所需要的核心知识点就是**IDEA插件开发**和**JCEF**，在本文中，我们将继续插件的开发，为该插件的JS Dialog显示进行自定义处理。

<!-- more -->

# 背景

在开发之前，我们首先要了解下什么是JS Dialog。有过Web页面开发经历的开发者都或多或少使用过这样一个JS的API：`alert('this is a message')`，当JS页面执行这段脚本的时候，在浏览器上会有类似于如下的显示：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/010-show-js-alert.gif)

同样，当我们使用`confirm('ok?')`的时候，会显示如下：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/020-show-js-confirm.gif)

以及，使用`prompt(input your name: ')`，有如下的显示：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/030-show-js-prompt.gif)



这些弹框一般来说都是原生的窗体，例如，当我们在之前的《IDEA Web渲染插件开发（一）》中的Web渲染插件来打开上面的Demo网页的时候，效果如下：

**alert**

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/040-show-js-alert-in-jcef.gif)

**confirm**

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/050-show-js-confirm-in-jcef.gif)

**prompt**

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/060-show-js-prompt-in-jcef.gif)

可以看到，原生窗体显得不是那么好看。那么，我们能不能自定义这个原生窗体呢？答案是肯定的，接下来就要用到**JCEF**里面一个Handler CefJSDialogHandler（[java-cef/CefJSDialogHandler](https://github.com/chromiumembedded/java-cef/blob/master/java/org/cef/handler/CefJSDialogHandler.java)）。

# CefJSDialogHandler

对于该Handler，官方注释为：

>Implement this interface to handle events related to JavaScript dialogs. The methods of this class will be called on the UI thread.
>
>实现此接口以处理与JavaScript对话框相关的事件。将在UI线程上调用此类的方法。

对于该Handler，里面有一个核心的接口方法：

```java
    /**
     * Called to run a JavaScript dialog. Set suppress_message to true and
     * return false to suppress the message (suppressing messages is preferable
     * to immediately executing the callback as this is used to detect presumably
     * malicious behavior like spamming alert messages in onbeforeunload). Set
     * suppress_message to false and return false to use the default
     * implementation (the default implementation will show one modal dialog at a
     * time and suppress any additional dialog requests until the displayed dialog
     * is dismissed). Return true if the application will use a custom dialog or
     * if the callback has been executed immediately. Custom dialogs may be either
     * modal or modeless. If a custom dialog is used the application must execute
     * callback once the custom dialog is dismissed.
     *
     * @param browser The corresponding browser.
     * @param origin_url The originating url.
     * @param dialog_type the dialog type.
     * @param message_text the text to be displayed.
     * @param default_prompt_text value will be specified for prompt dialogs only.
     * @param callback execute callback once the custom dialog is dismissed.
     * @param suppress_message set to true to suppress displaying the message.
     * @return false to use the default dialog implementation. Return true if the
     * application will use a custom dialog.
     */
    public boolean onJSDialog(CefBrowser browser, String origin_url, JSDialogType dialog_type,
            String message_text, String default_prompt_text, CefJSDialogCallback callback,
            BoolRef suppress_message);
```

注释翻译如下：

>在调用一个JS的Dialog的时候会调用该方法。设置`suppress_message`为`true`并使该方法返回`false`来抑制这个消息（抑制消息比立即执行回调更可取，因为它用于检测可能的恶意行为，如onbeforeunload中的垃圾邮件警报消息）。设置`suppress_message`为`false`并且返回`false`来使用默认的实现（默认的实现将会立刻展示一个模态对话框并抑制任何额外的对话框请求直到当前展示的对话框已经销毁）。如果应用程序想要使用一个自定义的对话框或是回调callback已经立刻被执行了，则返回`true`。自定义的对话框可以是模态或是非模态的。如果使用了一个自定义的对话框，那么一旦自定义对话框销毁后，应用程序需要立即执行回调。

首先，我们编写类JsDialogHandler，实现该接口：

```java
package com.compilemind.demo.handler;

import org.cef.browser.CefBrowser;
import org.cef.callback.CefJSDialogCallback;
import org.cef.handler.CefJSDialogHandler;
import org.cef.misc.BoolRef;

import static org.cef.handler.CefJSDialogHandler.JSDialogType.*;

public class JsDialogHandler implements CefJSDialogHandler {
    
    @Override
    public boolean onJSDialog(CefBrowser browser,
                              java.lang.String origin_url,
                              CefJSDialogHandler.JSDialogType dialog_type,
                              java.lang.String message_text,
                              java.lang.String default_prompt_text,
                              CefJSDialogCallback callback,
                              BoolRef suppress_message) {
        // 具体内容见下文
    }

    @Override
    public boolean onBeforeUnloadDialog(CefBrowser cefBrowser, String s, boolean b, CefJSDialogCallback cefJSDialogCallback) {
        return false;
    }

    @Override
    public void onResetDialogState(CefBrowser cefBrowser) {

    }

    @Override
    public void onDialogClosed(CefBrowser cefBrowser) {

    }
}

```

除了`onJSDialog`方法，其他的我们暂时不关心，使用默认的处理。对于`onJSDialog`的方法，我们编写如下的内容：

```java
    @Override
    public boolean onJSDialog(CefBrowser browser,
                              java.lang.String origin_url,
                              CefJSDialogHandler.JSDialogType dialog_type,
                              java.lang.String message_text,
                              java.lang.String default_prompt_text,
                              CefJSDialogCallback callback,
                              BoolRef suppress_message) {
        // 不抑制消息
        suppress_message.set(false);
        if (dialog_type == JSDIALOGTYPE_ALERT) {
            // alert 对话框

        } else if (dialog_type == JSDIALOGTYPE_CONFIRM) {
            // confirm 对话框
            
        } else if (dialog_type == JSDIALOGTYPE_PROMPT) {
            // prompt 对话框
            
        } else {
            // 默认处理，不过理论不会进入这一步
            return false;
        }

        // 返回true，表明自行处理
        return false;
    }
```

接下来，我们向CefBrowser进行注册（MyWebToolWindowContent类的构造函数中）：

```java
// 创建 JBCefBrowser
JBCefBrowser jbCefBrowser = new JBCefBrowser();
// 注册我们的Handler
jbCefBrowser.getJBCefClient()
        .addJSDialogHandler(
                new JsDialogHandler(),
                jbCefBrowser.getCefBrowser());
// 将 JBCefBrowser 的UI控件设置到Panel中
this.content.add(jbCefBrowser.getComponent(), BorderLayout.CENTER);
```

至此，我们已经在该方法中对js的对话框类型进行了区分。接下来，就需要我们针对不同的对话框类型，展示不同的UI，那么需要我们了解如何在IDEA插件中弹出对话框。

# IDEA插件对话框

## DialogWrapper

DialogWrapper是IntelliJ下的所有对话框的基类，他并不是一个实际的UI控件，而是一个抽象类，在调用其show方法的时候，由IntelliJ框架进行展示。

[Dialogs | IntelliJ Platform Plugin SDK (jetbrains.com)](https://plugins.jetbrains.com/docs/intellij/dialog-wrapper.html)

我们需要做的就是编写一个类来继承该Wrapper。

## AlertDialog

为了实现JS中的alert效果，我们首先编写AlertDialog：

```java
import com.intellij.openapi.ui.DialogWrapper;
import org.jetbrains.annotations.Nullable;

import javax.swing.*;

public class AlertDialog extends DialogWrapper {

    private final String content;

    public AlertDialog(String title, String content) {
        super(false);
        setTitle(title);
        this.content = content;
        // init方法需要在所有的值设置到位的时候才进行调用
        init();
    }

    @Override
    protected @Nullable JComponent createCenterPanel() {
        return new JLabel(this.content);
    }

}
```

这个Dialog的实现非常的简单，通过构造函数传入对话框的title和content。其中，title在构造函数执行的时候，就通过`DialogWrapper.setTitle(string)`完成设置；content赋值给AlertDialog的私有变量content，之后调用`DialogWrapper.init()`方法进行初始化。

**这里需要特别说明的是**，init方法最好放在Dialog的私有变量赋值保存完成后才进行，因为init方法内部就会调用下面重写的`createCenterPanel`方法。**如果没有这样做**，而是先`init()`，再进行`this.content = content`赋值，那么初始化的时候流程就是：

1. 设置title。
2. 调用init()。
3. Init()内部调用`createCenterPanel()`。
4. createCenterPanel返回一个空白的JLabel，因为此时`this.content`还是null。
5. 进行`this.content = content`赋值操作。

最终弹出的对话框效果就是没有任何的内容，本人在这里也是踩了坑。

AlertDialog编写完成后，我们可以在需要的地方编写如下的代码进行弹框展示：

```java
new AlertDialog("注意", "这是一个弹出框").show();
// 或
boolean isOk = new AlertDialog("注意", "这是一个弹出框").showAndGet();
```

于是，我们在之前的JSDialogHandler.onJSDialog中处理`dialog_type == JSDIALOGTYPE_ALERT`的场景：

```java
@Override
public boolean onJSDialog(CefBrowser browser,
                          java.lang.String origin_url,
                          CefJSDialogHandler.JSDialogType dialog_type,
                          java.lang.String message_text,
                          java.lang.String default_prompt_text,
                          CefJSDialogCallback callback,
                          BoolRef suppress_message) {
    // 不抑制消息
    suppress_message.set(false);
    if (dialog_type == JSDIALOGTYPE_ALERT) {
        // alert 对话框
        new AlertDialog("注意", message_text).show();
        return true;
    }
    return false;
}
```

### 问题处理

调试插件，当JS执行alert的时候，发现依然还是原生窗体。经过排查还会发现，**问题情况**如下：

- JS的alert依然是原生窗体。
- onJSDialog方法也进入了（可以使用断点或是控制台输出确认）。
- 控制台有异常：`Exception in thread "AWT-AppKit"`。

对于控制台的异常，详细如下：

```
Exception in thread "AWT-AppKit" com.intellij.openapi.diagnostic.RuntimeExceptionWithAttachments: EventQueue.isDispatchThread()=false Toolkit.getEventQueue()=com.intellij.ide.IdeEventQueue@fa771e7
```

对于EventQueue关键字的异常，有过GUI开发的读者应该很容易联想到应该是窗体事件消息机制的问题。

简单来说，**窗体GUI的线程一般都是独立的**，在这个线程中，会启动一个GUI事件队列循环，外部GUI输入（点击、拖动等等）会不断产生GUI事件对象，并按照一定的顺序进入事件循环队列，事件循环框架不断处理队列中的事件。对GUI的操作，比如修改窗体某个控件的文本或是想要对一个窗体进行模态显示，都需要在窗体GUI主线程进行，否则就会出现GUI的处理异常。

对于这类情况**最常见问题场景**就是：在窗体中点击一个按钮，点击后会单开一个线程异步加载大数据，加载完成后显示在窗体上。如果直接在加载大数据的线程中调用`Form.setBigData()`（假如有这样一个设置文本的方法），一般来说就会出现异常：**在非GUI线程中尝试修改GUI的相关值**。在Java AWT中解决的方式，调用`EventQueue.invokeLater(() -> { // do something} )`（异步）或是`EventQueue.invokeAndWait(() -> { // do something} )`（同步）。调用之后，`do something`就会被事件框架送入GUI线程执行了。

现在，我们回到一开始的问题，我们重新修改代码：

```java
if (dialog_type == JSDIALOGTYPE_ALERT) {
    // alert 对话框
    EventQueue.invokeLater(() -> {
      new AlertDialog("注意", message_text).show();
      callback.Continue(true, "");
    });
    return true;
}
```

我们对代码进行断点确认线程，在onJSDialog执行的时候，所运行的线程是：`AWT-AppKit`。

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/070-onJSDialog-thread.jpg)

而EventQueue.invokeLater中所运行的线程是：`AWT-EventQueue-0`，这个线程就是IDEA插件中的GUI线程。

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/080-EventQueueInvokeLater-thread.jpg)

修改线程处理后，让我们再次调用alert：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/090-new-not-perfect-alert-dialog.gif)

可以看到对话框已经显示为了使用IDEA插件下的dialog形式，但是这个dialog还不完全正确，一般的alert对话框，只会有一个确认按钮，而IDEA下的dialog默认是Cancel+OK的按钮组合。

### Dialog按钮自定义（重写createActions）

IDEA插件的DialogWrapper默认情况下是Cancel+OK的按钮组合。那么如何自定义我们的按钮呢？可行的一种方式就是重写createActions。这个方法需要我们返回实现`javax.swing.Action`接口的实例的数组，当然，IDEA插件也有对应的Wrapper：DialogWrapperAction。我们编写我们自己的OkAction：

```java
    protected class OkAction extends DialogWrapperAction {

        public OkAction() {
            super("确定");
        }

        @Override
        protected void doAction(ActionEvent e) {
            close(OK_EXIT_CODE);
        }
    }
```

**务必注意，DialogWrapperAction的实现子类，必须是DialogWrapper的内部类，否则无法查看。**

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/100-ok-action-in-alert.jpg)

重新运行，查看AlertDialog的效果：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/110-new-perfect-alert-dialog.gif)

接下来，我们需要编写ConfirmDialog，来处理JS中的confirm。

## ConfirmDialog

由于confirm天生需要取消和确定按钮，所以我们可以直接使用默认的DialogWrapper，不用重写Action的返回：

```java
import com.intellij.openapi.ui.DialogWrapper;
import org.jetbrains.annotations.Nullable;

import javax.swing.*;

public class ConfirmDialog extends DialogWrapper {

    private final String content;

    public ConfirmDialog(String title, String content) {
        super(false);
        setTitle(title);
        this.content = content;
        // init方法需要在所有的值设置到位的时候才进行调用
        init();
    }

    @Override
    protected @Nullable JComponent createCenterPanel() {
        return new JLabel(this.content);
    }

}
```

在Handler中，我们对`JSDIALOGTYPE_CONFIRM`分支进行：

```java
if (dialog_type == JSDIALOGTYPE_CONFIRM) {
    // confirm 对话框
    EventQueue.invokeLater(() -> {
        boolean isOk = new ConfirmDialog("注意", message_text).showAndGet();
        callback.Continue(isOk, "");
    });
    return true;
}
```

这点和AlertDialog的差别在于，需要调用`showAndGet`方法获取用户的点击是cancel还是ok的结果，使用callback返回给JS，才能使得JS的confirm调用获得正确的返回。下面是效果：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/120-new-confirm-dialog.gif)

## PromptDialog

对于PromptDialog，在对话框的界面，需要两个元素：**文本提示**和**文本输入**。同时，在对话框点击结束后，还需要获取用户的输入，代码如下：

```java
public class PromptDialog extends DialogWrapper {

    /**
     * 显示信息
     */
    private final String content;

    /**
     * 文本输入框
     */
    private final JTextField jTextField;

    public PromptDialog(String title, String content) {
        super(false);

        this.jTextField = new JTextField(10);
        this.content = content;

        setTitle(title);
        // init方法需要在所有的值设置到位的时候才进行调用
        init();
    }

    @Override
    protected @Nullable JComponent createCenterPanel() {
        // 2行1列的结构
        JPanel jPanel = new JPanel(new GridLayout(2, 1));
        jPanel.add(new JLabel(this.content));
        jPanel.add(this.jTextField);
        return jPanel;
    }

    public String getText() {
        return this.jTextField.getText();
    }
}
```

在这个类中，我们定义了一个私有字段`JTextField`，之所以需要在类中持有该引用，是因为我们定义一个方法`getText`，以便在对话框结束时，可以通过调用`PromptDialog.getText`来获取用户输入。

编写完成后，我们在onJSDialog中对prompt类型的对话框进行处理：

```java
if (dialog_type == JSDIALOGTYPE_PROMPT) {
    // prompt 对话框
    EventQueue.invokeLater(() -> {
        PromptDialog promptDialog = new PromptDialog("注意", message_text);
        boolean isOk = promptDialog.showAndGet();
        String text = promptDialog.getText();
        callback.Continue(isOk, text);
    });
    return true;
}
```

和之前不太一样的是，这里需要在showAndGet之后，调用getText来获取用户输入，并在`callback.Continue(isOk, text)方法中`传入用户的数据数据。最终效果如下：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-10-06-intelliJ-plugin-dev-2/130-new-prompt-dialog.gif)

# 源码

[w4ngzhen/intellij-jcef-plugin (github.com)](https://github.com/w4ngzhen/intellij-jcef-plugin)

本次相关代码提交：[support JsDialog](https://github.com/w4ngzhen/intellij-jcef-plugin/commit/7df78f33845db89f46e33663967f9c5780cb5dca)

