---
layout: post
title: Spring Bean装配笔记
date: 2018-03-10
tags: 
- Spring
- Bean
categories: 
- 技术
---

Spring中的Bean是一个很重要的概念。Spring作为一个Bean容器，它可以管理对象和对象之间的依赖关系，我们不需要自己建立对象，把这部分工作全部转交给容器完成，具有低耦合，对代码没有侵略性。

<!-- more -->

目前本人的水平还不足以透彻的分析Spring是如何来构建Bean的装配的概念的，这里仅来记录总结Spring中多种Bean的装配方式。

Spring的配置方式有Java类配置形式与XML配置两种形式。大致提一下，不赘述，如下：
```java
// Java类config形式，需要@Configuration注解来声明
@Configuration
public class MyConfig {
}
// 类路径下的XML形式
<?xml version="1.0" encoding="UTF-8"?>
<beans ...>
  ...
</beans>
```
Spring提供了三种装配机制：
```
隐式的bean发现机制和自动装配
在Java中进行显示配置
在XML中进行显示配置
```
**隐式的bean发现机制和自动装配**

**隐式bean发现与自动装配即我们完全不必在配置类或者配置文件中定义bean属性（但是这两者必要提供一种）。** 如何定义一个Java类是bean？我们可以在类上使用@Component注解声明一个bean。
首先定义一个接口：Playable
```java
public interface Playable {
    void play();
}
```
接下来，我们定一个CD类，实现该接口，并添加@Component注解
```java
@Component
public class CD implements Playable {
    public void play() {
        System.out.println("CD is playing...");
    }
}
```
如此定义，即可表明CD是一个bean。接下来如何开启扫描？

如果使用Java类配置的方式来配置Spring，我们可以使用@ComponentScan注解在配置类上，告诉Spring开启了组建扫描。并且进行形如如下的bean配置：
```java
@Configuration
// 务必注意此处的自动扫描的基础包
// 因为通常配置类与其他类的包在不同的地方
// 默认的包路径是当前配置类所在包下
@ComponentScan(backPakages = "xxx")
public class MyConfig {
}
```
如果使用XMl来定义，则如下定义：
```xml
<beans>
  <context:component-scan/>
</beans
```
接下来编写测试类：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = MyConfig.class)
public class MyConfigTest {
    @Autowired
    Playable playable; // 运行时，将扫描到的在容器中的CD bean装配到这里的playable
    @Test
    public void checkNULL() {
        Assert.assertNotNull(playable);
    }
}
// 结果自然是PASS
```

这里在配置类中我们完全没有定义任何的bean，Spring自动为我们扫描出对应的组件进行自动装配（Autowired）到测试类中。

CD这个类过于简单了，现在重构CD，使其拥有一个title属性，用以表示CD的名字。如下：
```java
@Component
public class CD implements Playable {
    private String title;

    public CD(String title) {
        this.title = title;
    }
    
    @Override
    public void play() {
        System.out.println("CD:" + title + " is playing...");
    }
}
```
如果我们再次运行测试代码，会发现报错：

Unsatisfied dependency expressed through constructor parameter 0;// 大意是提示我们，CD这个类没有默认的参数个数为0的构造器。这里我们大致可以猜测，Spring在进行普通的构建对象时，是调用的该类的默认构造函数，在Java中我们知道，在一个类中若定义了任意形式的构造函数，原先的默认无参构造函数自然失效，而我们定义了有参数的构造函数，所以这里Spring调用不了无参构造函数，顾不能为我们构造这个bean。解决办法就是添加一个无参构造函数。

但是问题还没有彻底解决，这里Spring只是为我们创建了一个title没有初始化的CD实例对象，我们应该如何去初始化这个title呢？注入的是一个普通的对象，我们同样可以使用@Autowired注入，但是这里只是一个字面量String，如何注入？其实我们可以采用一种更加直观的方式来注入——**显式配置**

**通过显式配置**

配置类形式如下：

```java
@Configuration
@ComponentScan(basePackages = "zhen")
public class MyConfig {
    @Bean
    Playable cd() {
        return new CD("JayChou");
    }
}
```
上面通过@Bean来声明下面我们将要定义一个bean，紧接着定义一个方法，返回值为Playable（这里没有严格的要求是接口类还是本身实现类，满足语法即可），方法名即为该bean对应的id，参数虽没有定义，但是不代表不能有，这里可以注入其它的bean。方法体中的内容自然是返回具体的实现类了，然而这里就很灵活，我们可以将字符串通过这里构造函数传入，假如我们的CD类中如果有setTitle方法，甚至还可以像下面这样：
```java
    @Bean
    Playable cd() {
        CD cd = new CD("hello");
        cd.setTitle("JayChou");
        return cd;
    }
```
配置文件XML如下：
```xml
<beans ...>
  <bean id="cd" class="zhen.CD">
    <constructor-arg value="JayChou" />
  </bean>
</beans>
```
对比上面的两种显示注入的用法，对应也是很清楚的，Java类中的方法名即为bean id，返回对象对应的类即为XML中的class属性。同时，根据构造函数参数类型的不同，也有不同的形式，这里本篇笔记不多提，以后会有相关的笔记探讨的。

但是请注意，如果Java类配置文件中已经有一个@Bean，同时还启动了自动扫描，在原先的组件类上添加了@Component注解，Spring是会我们创建两个同为CD类实例bean的。如下：
```java
// 注意已经添加了@Component注解
@Component
public class CD implements Playable {
    private String title = "Default Title";
    //  定义一个可以设置的title的方法
    public void setTitle(String title) {
        this.title = title;
    }
    
    @Override
    public void play() {
        System.out.println("CD:" + title + " is playing...");
    }
}

//配置类中开启了组建扫描，且也定义了一个bean
@Configuration
@ComponentScan(basePackages = "zhen")
public class MyConfig {
    @Bean
    Playable cd() {
        CD cd = new CD();
        cd.setTitle("JayChou");
        return cd;
    }
}
```
在测试类不变的情况下，运行测试代码，会报错：

No qualifying bean of type 'zhen.component.Playable' available: expected single matching bean but found 2: CD,cd

这行报错告诉我们，发现了两个bean：CD cd都满足Playable，都可以注入到此处。 **（注意，如果一个类有@Component组件注解，该类的bean id默认为类名首字母小写，这里CD由于其本身两个字母都是大写，Spring所以给其的默认名没有将首字母小写，一定注意命名特殊性）** 到底该选谁呢？如果我们不指定，Spring也不能为我们做主。如果我们将测试类中的Playable playable改为Playable cd 或者 CD，如下：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = MyConfig.class)
public class MyConfigTest {
    @Autowired
    // 为了配合下面输出title进行验证
    // 这里我将类型改为了CD，且CD类中的私有title我修改为了公有
    CD cd;
    // CD CD;
    @Test
    public void checkNULL() {
        System.out.println(cd.title); // 输出 “CD:JayChou is playing...”
        // System.out.println(CD.title); // 输出 "CD:Default String is playing..."
        
        // 下面均通过
        Assert.assertNotNull(cd);
        // Assert.assertNotNull(CD);
    }
}
```
由上面的现象，我们可以推测Spring在注入的时候，首先根据@Autowired下面的引用名来查找对应类型的bean id，如果没有，再找同类型的bean（这里要补充一下，@Autowired是根据类型来匹配注入的）

以上笔记大致总结了一下Spring bean装配问题，然而还有问题没有解决：
```
上面提到过的报错，Spring如果找到了多个同类型的bean会提示报错，在没有声明的情况下，Spring不知道到底改选哪一个，这就是自动装配的歧义性问题。
```
以上两个问题，我会继续做笔记的。

