---
title: 极简SpringBoot指南-Chapter03-基于SpringBoot的Web服务
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

# Chapter03-基于SpringBoot的Web服务

<!-- more -->

## 前言

终于，我们的教程来到了SpringBoot核心功能的介绍。对于本章，各位读者至少具备以下的知识：

1. maven的使用
2. HTTP

## SpringBoot的maven配置介绍

对于一个**简单的maven结构**的项目来说，以下结构：

```
${basedir}
    |-- pom.xml
    |-- src
        |-- main
            |-- java
                || com.xxx.xxx 项目源码
            |-- resources
                || 项目配置文件 .xml等
        |-- test
            |-- java
            |-- resources
```

在我们前面的教程中，我们都是在编写着代码，从未关注过maven的pom.xml究竟是如何配置的，因为项目脚手架已经配置完成了，也许有细心的同学已经看过了pom.xml文件的内容了：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

	<!-- ... 省略了当前项目本身信息的配置，如有需要，请读者查看源码 -->
    
    <!-- 当前pom的父级 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <dependencies>
		<!-- springboot框架web功能starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
		<!-- springboot框架的测试依赖starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- 构建流程使用SpringBoot的maven插件 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

对于一个普通的maven项目，其实核心的就是`dependencies节点-dependency节点`了。通过`dependency`节点，我们可以GAV坐标（Group-Artifact-Version）引入不同的库jar包，便于我们的项目使用这些库。

那么为什么在上面的pom出现了一个parent节点呢？实际上，pom允许所谓的继承：我们可以把一堆的pom依赖和配置，放到一个公共的pom里面，然后各个项目通过parent节点去引用这个公共的pom文件。由于SpringBoot并不是一个单一的jar包构成的框架，它的内部其实依赖了下面的核心库：

```
spring-core
spring-beans
spring-context
...还有很多
```

如果手动一项又一项编写dependency来使用Spring框架，不仅仅容易遗漏，而且十分不方便进行这些依赖库的版本管理。基于此，SpringBoot官方提供了父级pom：

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.3</version> <!-- 2021/08/09最新 -->
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
```

那么再看我们的pom文件中，还依赖了`spring-boot-starter-web`，这又是什么呢？难道有一个jar包叫做`spring-boot-starter-web`吗？其实不然。我们上面提到了parent POM，但是Spring框架下的依赖包特别多，并且有些包是核心的包，有些包则是在某些功能需要的情况下才依赖的包。如果一股脑的全部通过parent引入会让你的项目依赖十分臃肿，所以Spring官方再次按照包的功能进行了一定的组合，形成了所谓的starter，如果你只是想做web API的服务开发，用`spring-boot-starter-web`就可以了，要是需要使用AOP（面向切面编程，作切面开发），加上`spring-boot-starter-aop`依赖即可。

## 一个简单的Controller

```java
// 使用注解 @RestController，表明当前类是一个基于REST 规范的HTTP API Controller
@RestController 
// @RequestMapping 注解放在Controller上，用于标记 HTTP url的路径入口，
// 例如 http://localhost:8080/hello/xxx 才会进入当前Controller
@RequestMapping("hello") 
public class HelloController {

    // @RequestMapping 注解放在Controller的里面的方法上，
    // 将会与Controller上的RequestMapping组合成："/hello/say"
    // method用于指示通过何种HTTP方法访问
    // 在程序启动后，我们可以使用GET方法访问：http://localhost:8080/hello/say
    @RequestMapping(value = "say", method = RequestMethod.GET)
    public String say() {
        return "hello, world";
    }

}
```

编写启动类，启动我们的SpringBoot程序：

```java
@SpringBootApplication
public class Chapter03App {

    public static void main(String[] args) {
        SpringApplication.run(Chapter03App.class, args);
    }

}
```

启动成功后，通过HTTP访问：

```
http://localhost:8080/hello/say
# 输出：hello, world
```

## 关于Controller需要注意的点

上面的例子中，我们使用注解`@RestController`来标记了我们的Controller类，会有初学者使用`@Controller`来标记Controller，让我们改成它试试：

```java
@Controller // 改成使用 @Controller注解，其他不变，再次访问，看看有什么问题
@RequestMapping("hello")
public class HelloController {
// ...
}
```

重启启动应用后再次访问对应地址。如果是在浏览器中，你会看到：

```
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Mon Aug 09 10:16:59 CST 2021
There was an unexpected error (type=Not Found, status=404).
```

`type=Not Found, status=404`，404！为什么会这样呢？

1. 如果使用`@Controller`标记，那么将使用SpringMVC架构（自行了解），如果对应的方法返回的是字符串，则这个字符串表明需要查找对应的视图（View）名称，并将对应的视图通过视图解析器（InternalResourceViewResolver）解析修改为前端web页面。
2. 如果使用`@RestController`注解Controller，则Controller中的方法无法返回jsp页面，或者html，配置的视图解析器不起作用，返回的内容就是return里的内容。

针对情况1，解决方法就是在方法的返回上加上注解：`@ResponseBody`：

```java
    @ResponseBody // 如果当前Controller被@Controller注解，又想返回字符串或其他原始数据
    @RequestMapping(value = "say", method = RequestMethod.GET)
    public String say() {
        return "hello, world";
    }
```

## Controller如何被加载的？

在之前的文章，我们已经介绍了SpringBoot是如何初始化Bean并且将其放在IOC容器的。我们提到了三种方式：1、`@Component`；2、Java配置类；3、XML配置。对于第2、3点，好像目前我们的样例中并没有做手动配置的事情。那么是不是我们的Controller已经被`@Component`标记了呢？

查看@RestController的定义：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller // 原来你RestController也是一个Controller注解啊！
@ResponseBody // 并且，已经添加了@ResponseBody
public @interface RestController {
    @AliasFor(
        annotation = Controller.class
    )
    String value() default "";
}
```

`@RestController`其实组合了`@Controller`和`@ResponseBody`两个注解了。我们再看看`@Controller`的定义：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component // 原来你Controller注解也组合了Component注解
public @interface Controller {
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";
}
```

原来`@Controller`也已经组合了`@Component`注解啊，难怪我们的定义的Controller能被加载。

## Controller如何处理HTTP请求的？

要回答这个问题，需要你有一定的关于Java Tomcat Web容器的知识。本系列主要是对SpringBoot的快速入门，不便于讲的过细。简单一点就是：

1. Tomcat是一个用Java编写的Web容器，可以接受HTTP请求。
2. SpringBoot内置了Tomcat容器，并且对于HTTP请求，转到了内部处理
3. 对于这些HTTP请求，根据请求的路径等上下文，匹配Bean容器中的Controller进行处理

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)
