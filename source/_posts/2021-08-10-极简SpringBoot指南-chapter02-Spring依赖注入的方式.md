---
title: 极简SpringBoot指南-Chapter02-Spring依赖注入的方式
date: 2021-08-10
tags:
 - Java
 - SpringBoot
categories:
  - 技术
  - 极简SpringBoot指南 
---

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)

# Chapter02-Spring依赖注入的方式

<!-- more -->

我们在**Chapter00—2.2节依赖注入**已经介绍了Spring的对象依赖注入的方式，在那个例子中，我们使用了字段的setter方法对字段进行了注入。在本章中，我们将介绍对象依赖注入的另外的方式，并提到一些关于依赖注入的注意点。

大致来说，依赖注入分为三种：

1. 属性setter方法注入
2. 字段注入
3. 构造函数注入

为了 接下来的示例做准备，我们按照如下的代码结构顺序编写：

1. 编写类Pen，表示一个笔类Pen
2. 编写类Box，表示一个用于装Pen的盒子类Box
3. 编写相关配置注入的代码
4. 使用Spring验证代码注入

OK，首先编写类Pen做准备：

```java
@Component
public class Pen {
    public Pen() {
        System.out.println("Pen 无参构造函数");
    }
}
```

对于该类，我们使用`@Component`将其标记为Bean（PS：这里为了本章的主题，我们就不使用上一章的配置类和XML来声明了）。

## 一 属性setter方法注入

编写类BoxA类：

```java
@Component
public class BoxA {

    private Pen pen;

    // 我们在属性setter方法上，使用@Autowire注解
    // 目的是希望Spring容器框架能够让帮助我们注入该属性
    @Autowired
    public void setPen(Pen pen) {
        this.pen = pen;
        System.out.println("setter函数注入：" + pen);
    }

    public void printPen() {
        System.out.println(pen == null ? "BoxA没有Pen" : "BoxA有Pen：" + pen);
    }

}
```

对于该BoxA类，我们同样使用`@Component`标记为了Bean。此外，我们为其添加了Pen类型的字段pen，并编写了setter方法。在该方法上，我们添加了`@Autowired`注解，表明我们希望类型为Pen的属性pen能够由Spring为我们注入进来。

那么现在有一个问题，这个Pen的实例，是怎么来的呢？不难思考出来，就是我们一开头准备的`@Component Pen`。

接下来编写验证代码：

```java
@SpringBootApplication
public class Chapter02App {
    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(Chapter02App.class, args);
        // 我们只获取了BoxA的实例
        Object boxA = context.getBean("boxA");
        if (boxA instanceof BoxA) {
            System.out.println("BoxA实例：" + box);
            ((BoxA) boxA).printPen(); // 却能够正确打印Pen实例
        }
    }
}
```

## 二 构造函数注入

编写BoxB类：

```java
@Component
public class BoxB {

    private Pen pen;

    // 在构造函数上添加@Autowired注解，并且构造函数入参有Pen
    @Autowired
    public BoxB(Pen pen) {
        this.pen = pen;
        System.out.println("BoxB 有参构造函数，注入Pen实例：" + pen);
    }

    public void printPen() {
        System.out.println(pen == null ? "BoxB没有Pen" : "BoxB有Pen：" + pen);
    }
}
```

对于该类，我们同样拥有Pen类型字段pen，但和setter方法注入不同的是，我们没有编写setter方法，而是显式编写了BoxB的构造函数，并且构造函数的入参就是Pen类的实例，然后在该构造函数上也同样添加了注解`@Autowired`。其意图其实和setter方法注入相同，都是希望Spring框架能够帮我们进行依赖注入。

对于验证的代码，我们只需要稍微修改以下：

```java
@SpringBootApplication
public class Chapter02App {
    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(Chapter02App.class, args);
        // 我们只获取了BoxB的实例
        Object boxB = context.getBean("boxB");
        if (boxB instanceof BoxB) {
            System.out.println("BoxB实例：" + boxB);
            ((BoxB) boxB).printPen(); // BoxB 能够正确打印Pen实例
        }
    }
}
```

## 三 字段注入

对于字段注入来说，就更简单了，BoxC：

```java
@Component
public class BoxC {

    @Autowired
    private Pen pen;

    public void printPen() {
        System.out.println(pen == null ? "BoxC没有Pen" : "BoxC有Pen：" + pen);
    }
}
```

没有构造函数，没有setter方法，只有~~无尽的怒火~~字段，然后字段上添加`@Autowired`注解。验证代码同上，不再赘述。

字段注入比起另外的两种两种方式简单的多，可能绝大多数的项目都会用这个字段，但本人将**字段注入**放在了第三个来讲，还是希望说一下字段注入的问题点。使用IDEA同学可能已经看到了，IDEA会提示：

```
Field injection is not recommended 
不推荐使用字段注入
```

字段注入这么方便，为什么说不推荐呢？主要有以下几点：

1. 基于字段的依赖注入在声明为final的字段上不起作用。
2. 会与SpringIOC容器框架紧密耦合。因为private字段的原因，想要编写单元测试，就必须依赖Spring测试框架，否则你无法手动注入（除了使用反射，但是那样不久太麻烦了吗？）。

字段注入的问题还有其他的问题，可以自行搜索：Spring不推荐字段注入。

当然，如果一个项目自始自终都是在Spring框架中运行，也没有所谓的需要脱离Spring框架的地方，字段注入也并非不可。

## 扩展阅读：依赖注入的注意点

我们在上文已经提到了三种依赖注入的方式。那么读者有没有想过依赖注入需要注意什么呢？

编写类TestA、TestB：

```java
// TestA类
@Component
public class TestA {

    // 字段注入
    @Autowired
    private TestB testB;

    public void printTestB() {
        System.out.println("printTestB: " + this.testB);
    }
}
// TestB类
@Component
public class TestB {

    @Autowired
    private TestA testA;

    public void printTestA() {
        System.out.println("printTestA: " + this.testA);
    }
}
```

注意TestA类与TestB类相互以字段注入的方式注入到了另一个类中。接着我们编写测试代码：

```java
@SpringBootApplication
public class Chapter01CycleTestApp {
    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(Chapter01CycleTestApp.class, args);
        TestA testA = context.getBean(TestA.class);
        TestB testB = context.getBean(TestB.class);
        System.out.println("从容器中获取的TestA实例：" + testA);
        testA.printTestB();

        System.out.println("从容器中获取的TestB实例：" + testB);
        testB.printTestA();
    }
}
```

那么，测试代码能够正常运行呢？思考一下，我们似乎陷入了循环依赖的场景了：

```
Spring容器创建TestA实例 -> 发现需要注入TestB实例 -> 创建TestB实例 -> 发现需要注入实例TestA -> 创建TestA实例 -> ... 
```

似乎陷入了一直循环的情景。可是实际运行的时候，却正确输出了：

```
...
从容器中获取的TestA实例：TestA@4248ed58
printTestB: TestB@712ca57b
从容器中获取的TestB实例：TestB@712ca57b
printTestA: TestA@4248ed58

PS: 为了简洁的显示，我删去了包名
```

为什么会这样呢？实际上，Spring在初始化Bean的时候，并不是傻乎乎的按照上述的逻辑进行的，而是按照如下的大致流程：

1. 准备创建TestA实例
2. 发现TestA依赖TestB
3. 查找TestB实例未果
4. 先继续创建TestA实例，但是**内部会标记该类实际还未注入依赖**
5. 准备创建TestB实例
6. 发现TestB实例依赖TestA实例
7. 查找TestA实例成功（只不过目前还没注入依赖而已），注入
8. 扫描所有还未完全完成初始化的Bean，发现TestA还未注入依赖TestB
9. 再次检查发现TestB已经有了，将其注入到TestA中

当然，这里只是简单的梳理一个流程，在Spring内部是很复杂的。

那么现在再来思考另外一个相互依赖注入的情况：构造函数注入。

```java
// TestA 构造函数注入
@Component
public class TestA {

    // 字段注入
//    @Autowired
    private TestB testB;

    @Autowired
    public TestA(TestB testB) {
        this.testB = testB;
    }

    public void printTestB() {
        System.out.println("printTestB: " + this.testB);
    }
}
// TestB构造函数注入
@Component
public class TestB {

//    @Autowired
    private TestA testA;

    // 构造函数注入
    @Autowired
    public TestB(TestA testA) {
        this.testA = testA;
    }

    public void printTestA() {
        System.out.println("printTestA: " + this.testA);
    }
}

```

现在我们通过构造函数注入，执行测试代码看看会发生什么：

```
The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  testA defined in file [xxx\chapter02\cycle\TestA.class]
↑     ↓
|  testB defined in file [xxx\chapter02\cycle\TestB.class]
└─────┘
```

”当前应用程序中某些beans出现了循环依赖“。至于原因，请搜索关键词：`Spring构造函数注入与setter注入`

## 本章小结

在本章中，我们了解了Spring依赖注入的三种方式，并提到了循环依赖在不同注入方式下的区别。

至此，关于学习SpringBoot前的需要有的知识我们算是完成了一个简单的介绍的，接下来的文章将会基于SpringBoot框架，构建我们的Web服务。

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)
