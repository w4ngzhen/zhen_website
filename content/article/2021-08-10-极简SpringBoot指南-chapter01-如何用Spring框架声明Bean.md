---
title: 极简SpringBoot指南-Chapter01-如何用Spring框架声明Bean
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

# Chapter01-如何用Spring框架声明Bean

<!-- more -->

## 前言

在上一章中，我们已经掌握了Spring最基本的使用方式：

1. 通过使用`@Component`注解标记我们的类作为一个Bean组件。
2. 使用Spring上下文对象从IOC容器中拿到Bean组件。

在这一章中，我们将进一步学习使用Spring框架声明Bean的三种方式：

1. `@Component`注解
2. XML配置声明
3. Java类配置声明（`@Configuration`）

*PS：Spring声明Bean的方式不止这三种，请自行搜索了解：Spring声明Bean的方式。*

## 一 使用`@Component`注解

`@Component`注解我们已经很熟悉了，这里还是需要简单提一下。创建类Apple：

```java
// 使用注解 @Component 标记，并且我们还自定义了Bean的名称
@Component("myApple") 
public class Apple {

    public Apple() {
        System.out.println("Apple 无参构造函数");
    }

}
```

获取并打印：

```java
@SpringBootApplication
public class Chapter01App {
    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(Chapter01App.class, args);
        // 注意BeanName我们填写的"myApple"，和 @Component("myApple") 对应
        Object myApple = context.getBean("myApple");
        System.out.println(myApple);
    }
}
```

## 二 Java类配置声明

Java类配置声明是指，我们使用某个Java作为Spring的配置，通过Java代码本身来定义Bean。这样做的好处我们后文会提到。

首先编写我们的Bean类：

```java
/**
 * Banana类，不需要使用任何注解
 */
public class Banana {
    public Banana() {
        System.out.println("Banana 无参构造函数");
    }
}
```

然后，我们编写一个**配置类**，其特征是在类上有注解`@Configuration`：

```java
@Configuration // 添加注解表明是一个Spring配置类
public class Chapter01Configuration {
    @Bean // 使用'@Bean'在方法上，表明我们声明了一个Bean
    public Banana myBanana() { // 方法名即为Bean的名称
        System.out.println("进入Banana Bean创建函数");
        return new Banana();
    }
}
```

最后，我们在启动类上添加注入，表明扫描该配置类：

```java
@SpringBootApplication
// '@Import' 注解用于指定我们需要的配置类
// 默认情况下，会自动扫描当前启动类所在包及其子包的Configuration，
// 所以不需要特别指定，这里主要是演示
@Import(Chapter01Configuration.class)
public class Chapter01App {
    public static void main(String[] args) {
        ConfigurableApplicationContext context =
                SpringApplication.run(Chapter01App.class, args);
        Object myBanana = context.getBean("myBanana");
        System.out.println(myBanana);
    }
}
```

在这个例子中，各位读者可能会比较绕，这里再整理以下思路。

首先，我们创建了一个普普通通的Java类Banana，对于这个类，我们没有用到任何有关Spring框架的东西。

其次，我们创建了一个配置类Chapter01Configuration，通过类名我们知道，Spring对于配置类的名称没有任何的限制，只要是合法的都行。这个类与Spring框架唯一的联系就是使用了注解`@Configuration`，意味着我们标记了这个类为一个Spring的配置类。

那么配置类能干什么呢？就是我们第三步的操作：我们在配置类中编写了一个方法`public Banana myBanana();`，并且在其方法上使用了注解`@Bean`，通过该注解，表明我们定义了一个Bean，这个Bean的名称就是方法名"myBanana"，这个Bean的类型就是Banana。

## 三 XML配置声明Bean（选读）

还记得上面的我们通过注解的形式来声明我们的Bean吗？实际上，初期的Spring的IOC容器的初始化，是通过配置文件进行的，尽管如今Spring以及SpringBoot项目绝大多数情况都使用的是**注解方式**以及**Java类配置**，但是使用XML的配置方式，我也希望大家能够了解。

首先，我们依然创建一个普通的Java类Coconut：

```java
public class Coconut {

    /**
     * 定义字段：重量，可以通过下面的getter和setter获取和设置该值
     */
    private int weight;

    public Coconut() {
        System.out.println("Coconut 无参构造函数");
    }

    public int getWeight() {
        return weight;
    }

    public void setWeight(int weight) {
        this.weight = weight;
    }
}
```

从这个类可以看到，我们没有使用任何的有关Spring相关的注解（尤其是`@Component`注解），但是注意，这个类我们稍微添加了些内容，增加了一个字段weight，用于记录Coconut的重量。那么接下来，Spring的IOC容器，如何初始化我们的Coconut实例呢？只需两步：

1. 创建一个xml文件，在xml文件中，使用Spring框架约定的格式定义Bean
2. 在启动类上，指定Spring框架加载该xml

**首先我们看步骤1，创建xml文件。**在`项目目录/src/main/resources`目录下，创建任意自定义的名称的XML文件，为了便于我们识别，本人创建的是：`chapter01_03_use_xml_beans.xml`，初始内容下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

</beans>
```

*PS：使用IDEA的时候，可以在相关文件夹上`右键 — New — XML Configuration File — Spring Config`来快速创建具有初始内容的xml。*

可以看到xml初始内容的根节点`<beans>`就可以猜测到我们可以在其中添加子节点`<bean>`。OK，我们来定义我们的bean：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- 定义了一个Bean，名称是myCoconut -->
    <bean name="myCoconut" class="com.compilemind.guide.chapter01._03_use_xml.Coconut">
        <!-- 对于Coconut的weight属性，我们设置它的初始值100 -->
        <property name="weight" value="100"/>
    </bean>

</beans>
```

其实，通过xml，我认为不需要赘述什么。我们首先使用节点`<bean>`来定义我们的Bean，指明了他的name，以及他是由哪个类来创建的。并且，我们还在其子节点`<propertiy>`定义了Coconut的weight的初始值。

现在，我们已经通过xml，配置好了我们的Bean。怎么让Spring框架识别并初始化我们的Bean呢？很简单，只需要使用`@ImportResource`来配置我们要加载的xml是哪个即可：

```java

@SpringBootApplication
// '@ImportResource' 注解用于加载我们指定的xml配置
@ImportResource("classpath:chapter01_03_use_xml_beans.xml")
public class Chapter01App {
    public static void main(String[] args) {
        
        ConfigurableApplicationContext context =
                SpringApplication.run(Chapter01App.class, args);
        
		Object myCoconut = context.getBean("myCoconut");
        if (myCoconut instanceof Coconut) {
            int weight = ((Coconut) myCoconut).getWeight();
            System.out.println("Coconut重量是：" + weight);
        }
        
    }
}
```

*PS：对于`classpath:xxx`的知识点，请自行了解，搜索关键字：`Spring classpath`。*

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)
