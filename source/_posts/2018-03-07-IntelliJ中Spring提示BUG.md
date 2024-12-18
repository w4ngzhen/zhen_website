---
layout: post
title: IntelliJ中Spring识别BUG
date: 2018-03-07
tags: 
- Spring
- IntelliJ
categories: 
- 技术
---

最近开始学习Spring，在看《Spring实战4th》3.3“处理自动装配的歧义性”那一部分时，书上提到（也从网上看到了类似的用法）:
通过在一个类上加注@Component以及@Qualifier("x")可以为其配置限定符来标识区分同一个接口下的不同实现类，用以在需要进行@Autowired自动装配的地方使用@Qualifier("x")来指定特定的实现类对象bean。

<!-- more -->

但是本人在练习过程中，IntelliJ通过上述方式识别不了，只能在@Bean处添加@Qualifier("x")，在后续的测试中才能识别。如下：

在Dessert的实现类Cake上加上@Component以及@Qualifier("Lovely")

```java
@Component
@Qualifier("Lovely")
public class Cake implements Dessert { }
```

然后是配置类

```java
@Configuration
public class JavaConfig {
    @Bean
    Dessert cake() {
        return new Cake();
    }

    @Bean
    Dessert iceCream() {
        return new IceCream();
    }
}
```

测试类中无法识别：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = JavaConfig.class)
public class PlateTest {
    @Autowired
    @Qualifier("Lovely") // IDEA 提示"Cannot find bean with qualifier 'Lovely' "
    Dessert dessert;

    @Test
    public void testNotNull() {
        Assert.assertNotNull(dessert);
    }
}
```

但是，测试能PASS，应该是IntelliJ中的识别BUG。虽然是一个BUG，但是个人的理解是推荐在配置Bean中，进行限定，而不是在每一个类处进行限定。如下在JavaConfig类中进行配置：

```java
@Bean
@Qualifier("Lovely")
Dessert cake() {
  return new Cake();
}
```
这样就不会出现报提示错。
**以上在运行过程均没有错误，只是IDE辅助出现了问题。**
