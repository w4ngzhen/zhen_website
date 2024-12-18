---
title: 极简SpringBoot指南-Chapter00-学习SpringBoot前的基本知识
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

# Chapter00 学习SpringBoot前的基本知识

<!-- more -->

## 一 反射

在Java中，我们可以通过反射（Reflection）来获取任何类的类型信息，其中最值得关注的就是Class类。通过Class类，我们拿到任意对象的相关上下文。

例如，我们有一个User类，代码如下：

```java
public class User {

    private String name;

    private int age;

    public User() {
        System.out.println("User 无参构造函数");
    }

    public User(String name, int age) {
        this.name = name;
        this.age = age;
        System.out.printf("User 有参构造函数：name = %s, age = %d%n", name, age);
    }
    // 忽略一堆的getter、setter
}
```

如下的代码中，我们可以获取类上的声明的字段和各种方法：

```java
Class<User> userClass = User.class;
System.out.

println(userClass.getCanonicalName());

// 1. 显示声明的字段
Field[] fields = userClass.getDeclaredFields();
System.out.

println("该类包含了如下的字段：");
for(
Field field :fields){
        System.out.

println(field);
}

// 2. 显示声明的方法
Method[] methods = userClass.getDeclaredMethods();
System.out.

println("该类包含了如下的方法：");
for(
Method method :methods){
        System.out.

println(method);
}
```

当然，我们同样可以拿到构造函数，并通过反射的方式进行实例创建：

```java
// 3. 展示类上的构造函数对象
Constructor<?>[] constructors = userClass.getConstructors();
for(
Constructor<?> constructor :constructors){
        System.out.

println("找到构造函数："+constructor);
}

// 4. 通过无参构造函数创建实例
Constructor<?> constructor = userClass.getConstructor();
Object user1 = constructor.newInstance();
System.out.

println(user1);

// 5. 通过有参构造函数创建实例
// 注意有参构造函数获取时，传入了参数的class对象
// 以及在newInstance的时候，需要传入实际的值
Constructor<?> constructor1 = userClass.getConstructor(String.class, int.class);
Object user2 = constructor1.newInstance("Mr.Hello", 18);
System.out.

println(user2);
```

在上述代码中，我们使用代码符号的方式获取的对应类的Class对象：

```java
Class<User> userClass = User.class;
```

这种情况下，我们必须能拿到User的符号。但更多的情况下，我们并没有类符号，反射允许我们通过类型的名称来进行：

```java
Class<?> userClass = Class.forName("com.compilemind.guide.chapter00.manual.User");
```

此外，我们还可以拿到类上的注解。首先我们定义一个注解：

```java

@Target(ElementType.TYPE) // 放在类型上
@Retention(RetentionPolicy.RUNTIME) // 运行时保留
public @interface UserInfo {

    String name();

    int age();
}
```

然后给User上添加这个注解：

```java

@UserInfo(name = "annotationName", age = 99) // 使用注解
public class User {
    // ...
    public User(String name, int age) {
        this.name = name;
        this.age = age;
        System.out.printf("User 有参构造函数：name = %s, age = %d%n", name, age);
    }
    // ...
}
```

利用反射，我们可以获取注解，并通过注解的方式，创建我们的User对象：

```java
// 1. 获取Class类对象
Class<?> userClass = Class.forName(
                "com.compilemind.guide.chapter00.manual.User");

// 2. 获取类上的 @UserInfo 注解
UserInfo userInfo = userClass.getAnnotation(UserInfo.class);
if(userInfo ==null){
        System.out.

println(userClass.getCanonicalName() +"不包含注解"+UserInfo .class +"，终止创建");
        return;
        }

// 3. 获取注解信息
String name = userInfo.name();
int age = userInfo.age();

// 4. 通过有参构造函数，结合注解上的上下文，创建实例
Object user = userClass.getConstructor(String.class, int.class).newInstance(name, age);
System.out.

println(user);
```

该部分可以在chapter00.manual包下查看源码。

## 二 Spring"构造"对象

可能读者会疑惑，为什么`1.反射`和`2.Spring"构造"对象`
会放在一起，实际上Spring的对象构造的底层就是使用反射进行的。接下来，让我们看看Spring中，如何"构造"一个对象。

编写一个新的示例UserEx：

```java

@Component // 使用注解，标记该类为一个组件
public class UserEx {
    public UserEx() {
        System.out.println("UserEx 无参构造函数");
    }
}
```

在这个UserEx中，我们在类上添加了注解`@Component`，标记该类为一个**组件**。然后创建一个新的类：`IocApp`，编写如下的代码：

```java
/**
 * "@SpringBootApplication" 标记是一个SpringBoot应用
 * 启动后，SpringBoot框架会去扫描当前包以及子包下（默认情况）的所有具有@Component标记的类，
 * 并通过反射的方式创建这个类的实例，存放在Spring的Bean容器中。
 */
@SpringBootApplication //
public class IocApp {
    public static void main(String[] args) {
        // 1. 启动
        ConfigurableApplicationContext context =
                SpringApplication.run(IocApp.class, args);
        // 2. 类符号获取
        System.out.println("通过类符号获取Bean");
        UserEx userEx = context.getBean(UserEx.class);
        System.out.println(userEx);

        // 3. 通过Bean的名称获取
        System.out.println("通过类符号获取Bean");
        UserEx userEx2 = (UserEx) context.getBean("userEx");
        System.out.println(userEx2);
    }
}
```

在这段代码中，在有main函数的类上，添加`@SpringBootApplication`，标记是一个**SpringBoot**应用。

接着，我们在main函数中调用`SpringApplication.run(IocApp.class, args);`
来启动这个SpringBoot应用。启动后，SpringBoot框架会去扫描当前包以及子包下（默认情况）的所有具有`@Component`
标记的类，并通过反射的方式创建这个类的实例，存放在Spring的**Bean**容器中。

最后，我们通过调用`ConfigurableApplicationContext.getBean`来获取实例并进行打印。在这里，我们使用了两种方式来获取Bean：

```java
// 传入类符号.class
<T> T getBean(Class<T> requiredType) throws BeansException;

// 传入Bean的名称
Object getBean(String name) throws BeansException;
```

这里简单提一下，Bean的name是有一定的规则。默认情况下，是类名称的小驼峰形式，这里UserEx对应的名称就是userEx；但是我们通过设置注解的name字段：
`@Component("myUserEx")`，能够自定义在Bean在容器中的名称。

看到这里，让我们再次回顾第一节反射中的操作：我们在`User`类上添加注解`@UserInfo`，然后通过反射，获取注解的信息，并创建User实例。

那么现在，各位读者能否将第二节上述的内容，和反射中的操作关联起来呢？如果你能够大致理解我现在讲的含义，那么恭喜你，对于Spring进行对象构建的内部原理你已经有了一个简单的认识。

### 2.1.IOC控制反转

细心的读者已经发现了，在上一节中，我们创建了包含启动代码的类：`IocApp`。没错，此IOC就是这一节要讲的IOC（Inversion of
Control），控制反转。

> 传统的创建对象的方法是直接通过 **new 关键字**，而 spring 则是通过 IOC 容器来创建对象，也就是说我们将创建对象的控制权交给了
> IOC 容器。我们可以用一句话来概括 IOC：
>
>**IOC 让程序员不在关注怎么去创建对象，而是关注与对象创建之后的操作，把对象的创建、初始化、销毁等工作交给spring容器来做。**

具体结合前面的例子来说，对于UserEx类，**在没有IOC思想的介入下**，我们创建这个UserEx类通常会这样做：

```
UserEx userEx = new UserEx();
// ... 使用该实例
```

而在Spring IOC框架下，我们做了如下的工作：

1. 编写UserEx类，标注`@Component`；
2. 初始化容器：`SpringApplication.run(...)`；
3. 从容器中获取：`UserEx userEx = context.getBean(UserEx.class);`

看到这里，你也许会想没有IOC的模式下，我只要一行代码，而现在使用了Spring的IOC，多了这么多的配置和操作。难道不是更加的麻烦了吗？对于这个例子，的确是这样的，但是仔细一想，随着项目的体积逐渐增大，越来越多的类实例需要被创建，难道那个时候我们还要如此繁杂的通过new创建一大堆实例吗？另外，IOC容器帮我们做的事情，还远远不止控制反转这一项，依赖注入（dependence
injection）也是一个重要的能力。

### 2.2.DI依赖注入

说到依赖注入，我们首先需要明确，在代码中什么是依赖。从互联网上有这样一段对于依赖的定义，我觉得很好：

> 每个软件，都是由很多“组件”构成的。这里的“组件”是指广义的组件 ——
> 组成部件，它可能是函数，可能是类，可能是包，也可能是微服务。软件的架构，就是组件以及组件之间的关系。而这些组件之间的关系，就是（广义的）依赖关系。

从狭义来讲，我们定义一个类GameEx，这GameEx包含前文的UserEx，我们就可以认为GameEx依赖UserEx：

```java
public class GameEx {

    private UserEx userEx; // GameEx依赖UserEx

    public void setUserEx(UserEx userEx) {
        this.userEx = userEx;
        System.out.println("调用setUserEx");
    }

    /**
     * 打印GameEx的UserEx
     */
    public void printUserEx() {
        if (this.userEx == null) {
            System.out.println("无用户");
        } else {
            System.out.println(this.userEx);
        }
    }

}
```

在上述GameEx类中，拥有一个UserEx类型的字段userEx。同时，包含一个名为`printUserEx`的方法，用以打印UserEx实例。为了**避免**得到输出
`"无用户"`，我们需要在调用该方法前，设置UserEx的实例：

```java
// 伪代码
UserEx userEx = ...; // 1.通过某种方式，获得的UserEx的实例
GameEx gameEx = ...; // 2.通过某种方式，获得的GameEx的实例
        gameEx.

setUserEx(userEx); // 3.调用setter方法将UserEx实例设置到GameEx中
gameEx.

printUserEx(); // 4.输出: UserEx@xxxx
```

在上面伪代码的第3步中，我们通过代码的方式手动调用setter函数。这个过程，本质上来讲，**就是我们在控制着依赖**
：因为我们明白GameEx的功能依赖于UserEx，所以为了达到预期的目的，我们需要手动进行代码编写，处理这样的依赖。

让我们再看上述伪代码的中的第1、2步：得到UserEx和GameEx实例。在第2节中，我们已经学会了如何使用Spring的IOC容器创建对象，所以对于GameEx类，我们同样可以增加注解
`@Component`，将GameEx标记为Bean，让Spring的IOC容器管理起来：

```java

@Component
public class GameEx {
    // 其他代码忽略 ...
}
```

然后，我们在IocApp中实现上述的伪代码效果：

```java

@SpringBootApplication //
public class IocApp {
    public static void main(String[] args) {
        // 1. 启动
        ConfigurableApplicationContext context =
                SpringApplication.run(IocApp.class, args);
        manualDependencySet(context);
    }

    /**
     * 第2.2节伪代码实现：手动进行依赖设置
     */
    private static void manualDependencySet(ConfigurableApplicationContext context) {
        UserEx userEx = context.getBean(UserEx.class); // 1.
        GameEx gameEx = context.getBean(GameEx.class); // 2.
        gameEx.setUserEx(userEx); // 3.
        gameEx.printUserEx(); // 4.
    }
}
```

进行执行后，我们可以从控制台中观察到相关输出：

```
UserEx 无参构造函数
// 其他log信息
调用setUserEx
com.compilemind.guide.chapter00.spring.UserEx@5300f14a
```

在上面的例子中，我们手动进行了依赖的管理，那么Spring的IOC容器是否可以帮助我们去管理依赖吗？答案是肯定的。我们只需要在需要注入依赖字段的setter方法上（后面会介绍其他的方式），加上
`@Autowired`注解即可实现这样的功能：

```java

@Component
public class GameEx {

    private UserEx userEx;

    @Autowired // 使用注解 @Autowired，表明希望IOC容器为我们注入这个UserEx的实例
    public void setUserEx(UserEx userEx) {
        this.userEx = userEx;
        System.out.println("调用setUserEx");
    }
    // 忽略其他代码
}
```

此时，我们不再需要分别从IOC容器中获取UserEx和GameEx来手动设置依赖，因为SpringIOC容器已经帮助我们完成了：

```java

@SpringBootApplication //
public class IocApp {
    public static void main(String[] args) {
        // 1. 启动
        ConfigurableApplicationContext context =
                SpringApplication.run(IocApp.class, args);
        dependencyInject(context);
    }

    /**
     * 不再需要手工设置了
     */
    private static void dependencyInject(ConfigurableApplicationContext context) {
        GameEx gameEx = context.getBean(GameEx.class);
        gameEx.printUserEx();
    }
}
```

最后的输出与上面手动设置依赖是相同。

## 本章总结

在本章中，我们了解了Java中关于反射的一些基础知识，了解了如何通过反射而不是new的形式创建对象。在此基础上，我们介绍了Spring
IOC容器，让大家明白了Spring IOC底层的基本原理。最后，我们介绍了IOC控制反转以及DI依赖注入的概念，并用Spring框架演示了这些概念。在下一章，我们将介绍使用SpringIOC容器来创建Bean的几种方式。

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)

