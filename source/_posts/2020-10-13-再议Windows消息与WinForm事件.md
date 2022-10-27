---

title: 再议Windows消息与WinForm事件
date: 2020-10-13
tags: 
- WinForm

---

# 前言

在2月份的时候，我之前曾经写过一篇关于Windows消息与C# WinForm事件机制的文章，名为《WinForm事件与消息》。在那篇文章中，我简单探讨了一下事件和消息。然而如今看来，当时的文章中的案例在运行上存在一定的问题，并且内容也有所缺陷，于是本文将重新优化文章的内容。

<!-- more -->

# 消息概述

Windows下窗体应用程序的执行是通过消息驱动的。所有的外部事件，如键盘输入、鼠标移动、按动鼠标都由用户所触发；然后OS接收到对应的“消息”；然后送入消息队列中；接下来，启动的应用程序的工作引擎通过轮询等方式遍历获取，然后按照消息的类型逐个分发（Dispatch）到对应的组件（例如窗体、按钮等），最后才调用对应组件所注册的事件进行处理。

# 处理消息

一般来说，使用WinForm技术进行开发，绝大部分的情况下，我们都在做上述流程的最后一件事情：给各种控件注册事件。毕竟，WinForm真的为我们封装了绝大部分的事件了。而通常的WinForm开发，我们都离不开一个东西：System.Windows.Forms.Application。

## System.Windows.Forms.Application

Application具有用于启动和停止应用程序和线程以及处理Windows消息的方法。例如，调用Run以启动当前线程上的应用程序消息循环，并可以选择使其窗体可见；调用Exit或ExitThread来停止消息循环。所以我们经常使用vs初始化一个基本的WinForm程序，显示的下列模板代码：

```C#
/// <summary>
/// 应用程序的主入口点。
/// </summary>
[STAThread]
static void Main()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    Application.Run(new Form1()); // 调用Run以启动当前线程上的应用程序消息循环
}
```

因为Application是在单线程中运行的，所以在Application.Run开始后，Application本身不断轮询检查消息队列，然后根据消息类型进行数据分发。例如，当我们为这个Form1增加一个鼠标的点击事件后，我们运行该打开Form1：

```C#
Form1 form1 = new Form1();
form1.MouseClick += 
                (sender, e) => MessageBox.Show(@"MouseClick 1");
form1.MouseClick += 
                (sender, e) => MessageBox.Show(@"MouseClick 2");
Application.Run(form1);
```

运行后点击Form，可以看到首先出现一个MessageBox，展示“MouseClick 1”，我们点击确定后，又会出现MessageBox，展示“MouseClick 2”。实际上整个过程应该如下：

当我们按下鼠标左键后，消息形成并送往应用程序消息队列中，然后被Application类从应用程序消息队列中取出，然后分发到相应的窗体。窗体使用MouseClick事件中的函数指针调用已经添加的响应函数。所以C#中的事件字段实质上是一个函数指针列表，用来维护一些消息到达时的响应函数的地址。

到目前为止我们可以看到，消息其实在我们进行事件调用的时候，已经被提取加工了，它已经由Application进行了预处理，形成了所谓的“事件调用”。那么，我们还能更加自定义的干预消息吗？答案是可以的。

## WndProc

在.NET框架类库中的System.Windows.Forms命名空间中微软采用面对对象的方式重新定义了Message。该消息主要有一下的几个公共属性：

```
System.Windows.Forms.Message
HWnd     获取或设定消息的处理函数
Msg      获取或设定消息的ID号
Lparam   指定消息的LParam字段
Wparam   指定消息的WParam字段
Result   指定为响应消息处理函数而向OS系统返回的值
```

### WndProc

```
//
 // 摘要:
//     处理 Windows 消息。
//
// 参数:
//   m:
//     要处理的 Windows System.Windows.Forms.Message。
protected override void WndProc(ref System.Windows.Forms.Message e);
```
对于每个Form来说，我们都可以重写该方法，该方法的参数就是上面提到的Message类的实例，所有的消息在被获取后，正常情况下都会被封装为Message对象，然后由Application工作引擎调用对用的Form.WndProc传入该Messsage，由于Form子类重写了该方法，所以如果希望底层能处理相关的消息，需要通过base.WndProc传递到父类继续调用。下面就是一个代码示例来展示控制如果当前的消息是鼠标左键点击，则弹出MessageBox展示“WndProc MouseClick”：

```c#
        protected override void WndProc(ref Message m)
        {
            const int WM_LBUTTONDOWN = 0x0201;// 鼠标左键点击
            if (m.Msg == WM_LBUTTONDOWN)
            {
                MessageBox.Show("WndProc MouseClick");
                return;
            }
            base.WndProc(ref m);
        }
```

### IMessageFilter

除了上述的WndProc之外，其实更加便于处理应该的实现IMessageFilter接口，然后让Application将实现该接口的消息过滤器添加到Application中：

```c#
   public class MyMessageFilter : IMessageFilter
    {
        public bool PreFilterMessage(ref Message m)
        {
            //返回值为true， 表示消息已被处理，不要再往后传递，因此消息被截获
            //返回值为false，表示消息未被处理，需要再往后传递，因此消息未被截获
            const int WM_LBUTTONDOWN = 0x0201;// 鼠标左键点击
            if (m.Msg == WM_LBUTTONDOWN)
            {
                MessageBox.Show("MyMessageFilter MouseClick");
                return true;
            }
            return false;
        }
    }
```

编写完成后，在应用程序初始化的过程中，添加该过滤器：

```C#
Application.AddMessageFilter(new MyMessageFilter());
```

同样的，我们启动应用程序并点击实验，可以看到正常的MessageBox输出。






