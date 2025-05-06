---
layout: post
title: Scala trait特质深入理解
date: 2018-04-24
tags: 
- Scala
- trait
categories: 
- 技术
---

### 初探Scala 特质trait
在Scala中，trait（特质）关键字有着举足轻重的作用。就像在Java中一样，我们只能在Scala中通过extends进行单一继承，但trait可以让我们从语义上实现了多重继承。通过对继承的类混入（mixin）多个特质，来达到多重继承的目的。乍一看，trait和Java中的interface接口很像，但是，细节上它们又有着大不同。

<!-- more -->

```scala
// 定义超类
class Super
// 定义特质
trait Trait
// 定义子类，将特质使用with关键字混入 Super 类，并继承之
class Sub extends Super with Trait
```
请务必体会上面的最后一条注释，早期学习的时候，我们往往容易将特质理解为Java中的接口，同时将with理解为Java中的implements，于是我们编写出如下的代码：
```java
class A with T // 错误✖️
```
这样想是因为我们没有正确理解特质，特质是对我们要**被继承**的类的补充，是要混入我们要继承的类的，不是我们本身类！即我们要明确，对于混入特质的子类定义，其实整体分为两个部分：
```
[ class A ] extends [ S with T1 with T2 ...] 
```
中括号的表示两个部分才是正确的结构！

此外，与Java中的接口相比，我们还能够在特质中直接实现完整的方法，就像如下：
```scala
trait TrMid1 {
  def f: Unit = println("In TrMid 1's f.")
}
```
在Java8以前我们无法在接口中定义并实现完整的方法，而在Java8及以后我们可以通过default关键字来后面接完整的方法来实现。

现在，我们定义两个如下的特质：
```scala
trait TrMid1 {
  def f: Unit = println("In TrMid 1's f.")
}
trait TrMid2 {
  def f: Unit = println("In TrMid 2's f.")
}
```
之后，我们定义一个名为Bottom的类，继承Scala中的Any类，同时混入上面定义的特质
```scala

class Bottom extends Any with TrMid1 with TrMid2
/* 
这里的Any是scala中的特殊对象
这里使用Any会报错
Error:(16, 31) illegal inheritance; superclass Any
 is not a subclass of the superclass Object
 of the mixin trait TrMid2
...
*/
```
“超类Any不是混入特质TrMid2的超类Object的子类”，即我们extends的类需要是Object的子类才行。面对上面的错误，我们可以将Any修改为AnyRef，这个类是scala内建类，本质上对应Java中的java.lang.Object类。修改之后，编译还是会报错：
```scala
...
class Bottom extends AnyRef with TrMid1 with TrMid2
...
/*
Error:(16, 7) class Bottom inherits conflicting members:
  method f in trait TrMid2 of type => Unit  and
  method f in trait TrMid1 of type => Unit
(Note: this can be resolved by declaring an override in class Bottom.)
class Bottom extends AnyRef with TrMid2 with TrMid1 {
*/
```
这个错误提示我们在上面的两个特质中，均定义了相同签名的方法f，这里编译不通过，建议我们在Bottom类中实现f方法。换言之，当混入的多个特质中，分别定义了同名的方法，那么Scala会编译报错。在Java中也有类似的错误情形，下面就是关于Java，当然你可以跳过这一部分

___
#### Java接口中的同名方法

以上的讨论，务必与Java中区分开来，这里需要补充一下。在Java中，首先由于我们无法进行多重继承，我们只能编写形如这样的class AA extends BB implements CC, DD这样的代码，想要建立像上面的测试情形，我们还只能使用JDK8之后的能够在接口中使用default关键字来定义具有具体实现的方法，最终我们的代码如下：
```java
// [abstract] 为可选
[abstract] class BB {
    public void f() {
        System.out.println("In BB's f()");
    }
}

interface CC {
    // JDK8及以后的default关键字
    default void f() {
        System.out.println("In CC's f()");
    }
}

// 测试类
class AA extends BB implements CC {}

// 对比验证类，保证我们接口中的方法没问题
class ZZ implements CC {}

public class Test {
    public static void main(String[] args) {
        new AA().f();
        // In BB's f()
        new ZZ().f();
        // In CC's f()，接口方法是没有问题的
    }
}
```
根据上面的区别，我们首先确定是类似这样的class AA extends BB implements CC，同时BB，CC有同名的方法，Java中首先（只会？）找类中的实现。可能你有疑惑了，为什么不能够让多个接口都创建同名的默认方法，让一个类来实现它们，调用同名方法，就想这样：class AA implements CC, DD，遗憾的是（高兴的是？），当你这样做的时候，Java编译期就给你报错了啦！
```java
interface CC {
    default void f() {
        System.out.println("In CC's f()");
    }
}
interface DD {
    default void f() {
        System.out.println("In DD's f()");
    }
}
class AA implements CC, DD {}
// 编译就会报错
// Error:(25, 1) java: 类 AA从类型 CC 和 DD 中继承了f() 的不相关默认值
```
要解决这个错误，要么，让其中一个接口中的同名方法改名，要么，实现类重写这个方法。

____

#### 继续Scala特质的讨论

上面的关于同名方法的报错曾提示我们，在底层类重写实现Bottom中的f方法。这是办法之一。但是，我们还有另一种方式。

```scala
// 首先我们定义一个顶层的抽象类
abstract class AbTop {
 def f: Unit
}
// 让TrMid1、TrMid2均继承AbTop这个顶层抽象类，同时均重写抽象类中的方法
trait TrMid1 extends AbTop {
  override def f: Unit = println("In TrMid 1's f.")
}
trait TrMid2 extends AbTop {
  override def f: Unit = println("In TrMid 2's f.")
}
// Bottom类继承AbTop类并混入上面定义的两个特质
class Bottom extends AbTop with TrMid1 with TrMid2 {}
// 进行测试
object TraitTest {
  def main(args: Array[String]): Unit = {
    val top: Bottom = new Bottom
    top.f
  }
}
```
我们编译运行这段程序，得到了如下的结果：
```scala
In TrMid 2's f.
```
首先我们可以确定，我们按照上面的类层级结构混入了两个带有同名方法f的特质，并没有像上面那样出现二义性错误；为什么会这样？让我们再次理解这一段话“特质是对我们要**被继承**的类的补充，是要混入我们要继承的类的，不是我们本身类”。也就是说，报错的那个二义性，是由于我们想要将两个同名的f方法混入AnyRef这个类中，然而，我们没有override关键字（也无法加上），那么混入过程只是单纯的向AnyRef类中添加两个签名一样的方法f，而语法上我们无法向同一个类中添加连个签名完全一样的方法，故报错；解决方法就是在我们的子类中override这个f方法，重写覆盖它，消除二选一。

而后者，我们能够编译运行还是像上面这样理解，由于我们是要向AbTop这个类中去混入特质，而我们每一个特质都是继承了AbTop类的，故我们应当重写覆盖顶层抽象类中的f方法，所以，在混入的过程中，从左到右每混入一次，他就加上一层外壳，所以这就是为什么，输出的结果是打印的第二个特质中f方法的输出，因为逐渐混入加壳的过程是从左到右的，先对AbTop加了壳，混入TrMid1特质，然后又对这一个结构加壳过程，混入TrMid2特质，就像下图：
![wrap](https://res.zhen.blog/images/post/2018-04-24-trait/wrap.png)
这样一来，不难理解混入特质的过程（加壳的过程）本身就像一个一层一层继承的过程。还是上面那段带有AbTop的代码中，这一次我们添加一个新的抽象类AbNewTop，但是其中包含一个抽象方法其名称依然为f，然我们修改Bottom的定义：
```scala
abstract class AbNewTop {
  def f: Unit
}
...
...
class Bottom extends AbNewTop with TrMid2 with TrMid1 {}
...
```
注意我们让Bottom不再继承AbTop，而是继承新定义的AbNewTop，其他诸如TrMid1依然继承的是AbTop不变。让我们运行代码，报错：
```
Error:(19, 36) illegal inheritance; superclass AbNewTop
 is not a subclass of the superclass AbTop
 of the mixin trait TrMid2
class Bottom extends AbNewTop with TrMid2 with TrMid1 {
```
英语有点绕口，我们这里翻译并分割一下：“非法的继承；_超类AbNewTop_ 不是 混入特质的TrMid2的 _超类AbTop_ 的子类”。再次对应这个结构：[ class A ] extends [ S with T1 with T2 ...] 那么错误就在与后面的 S 与 T1、T2 对应不上了，及要实现正确的混入，S必须是T1、T2的超类的子类，当然，隐含的，本身也可以。转化为类图应该要满足如下的情形：
![class_map](https://res.zhen.blog/images/post/2018-04-24-trait/class_map.png)
可能有些人有疑惑，为什么特质不继承自任何其他的类的时候，依然可以被混入到其他的类中，就像如下的形式：
```scala
trait T1 {}
class Animal {}
class Dog extends Animal with T1 {}
```
因为在Scala任何的非值类（或特质）有默认的继承了scala.AnyRef类！这里的类图是如下的情形：
![animal_class](https://res.zhen.blog/images/post/2018-04-24-trait/animal_class.png)

### 使用特质来做可堆叠的改动——过滤

试想一种情形：现在有一个客户需要我们编写一种整形数字容器，这个容器的类似于Java中的List，我们可以往里面去添加数据，但是添加数据的过程是可过滤的，或者说可按条件进行预处理的。首先我们定义一个Container类：
```scala
class Container  {
  private val list: util.ArrayList[Int] = new util.ArrayList[Int]()
  def add(n: Int): Unit = list.add(n)
  def get(idx: Int): Int = list.get(idx)
}
```
目前为止，这个类似乎没什么特殊之处，甚至可以说是多此一举 —— 定义Container类的结构还不如直接使用一个util.ArrayList来的快。但是，试想一个场景，我现在要在添加之前 想要 首先检查这个是是否是偶数，如果不是，直接丢弃；如果是，则除以2再加入到容器中。如果仅仅使用现在的版本，你可能会直接在Container.add方法中去进行筛选，就像如下：
```scala
class Container extends AbstractContainer {
...
  override def add(n: Int): Unit = if (n % 2 == 0) list.add(n / 2)
...
}
```
诚然，这段简单的代码的确能工作的很好。但是每一次的我改变规则，你难道就要在这个add代码中进行修改吗？亦或者假设这个类的源码根本就无法修改。你又如何操作？于是，使用特质来堆叠能够发挥作用：
首先我们还原add代码为最初始的状态，然后，我们定义如下的两个特质，并且定义我们的MyContainer类继承Container类并混入这两个特质：
```scala
trait Even extends Container {
  override def add(x: Int): Unit = if (x % 2 == 0) super.add(x)
}

trait Divide extends Container {
  override def add(x: Int): Unit = super.add(x / 2)
}

class MyContainer extends Container with Divide with Even
// 测试这段代码
object Run {
  def main(args: Array[String]): Unit = {
    val myContainer = new MyContainer
    myContainer.add(1)
    myContainer.add(6)
    println(myContainer.get(0)) // 输出3
    // println(myContainer.get(1)) // 超出边界
  }
}
```
在前面的讨论中我们知道，如果是一个类混入了多个特质，这多个特质含有同名的方法，会从左到右包装出来，即最终调用的是靠近右侧的实现了的方法。首先要实现筛选偶数，再除以2，最终添加到容器中。所以最先发挥作用的Even特质放在了最右侧。为什么这里，不仅能够筛选出偶数，同时还能除以2呢？答案就在super这个关键点。super.add即调用超类的add方法。这里再次用图来说明：
![evendivide](https://res.zhen.blog/images/post/2018-04-24-trait/evendivide.png)
我想这个图足以说明了吧。调用过程就是先调用最右侧的Even.add方法，进行偶数筛选；然后调用超类super.add(x)；超类即从右到左开始Even左侧是Divide，Divide.add(x)，Divide.add内部对x除以2，传入super.add()方法，即再次向左侧，是Container.add()，此时接收到的数已经是除以2的数了：
![flow](https://res.zhen.blog/images/post/2018-04-24-trait/flow.png)
还有一种情况是更为复杂的：
```scala
class A {
  def f: Unit = println("In A")
}
trait B extends A {
  override def f: Unit = {
    println("In B")
    super.f
  }
}
trait C extends B {
  override def f: Unit = {
    println("In C")
    super.f
  }
}
class T extends A with C
```
关系图对应如下：
![classrelation](https://res.zhen.blog/images/post/2018-04-24-trait/classrelation.png)
输出：
```scala
object Run {
  def main(args: Array[String]): Unit = {
    val t: T = new T
    t.f
  }
}
// 输出
In C
In B
In A
```
看到输出，再根据前面的内容，我们也很容易理解，当某一个特质本生继承了其他的类的时候，super一定是其对应的超类，而不是class AA extends BB with TT中的BB这个类！

