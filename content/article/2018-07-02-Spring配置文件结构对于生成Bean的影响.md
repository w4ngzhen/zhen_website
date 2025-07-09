---
layout: post
title: Spring配置文件结构对于生成Bean的影响
date: 2018-07-02
tags: 
- Spring
- Bean
categories:
  - 技术
---

由于前段时间忙于毕设，导致Spring学习的东西忘了很多，所以最近又开始从头看Spring的基础。基础的Bean的装配不再多说了。这一次，主要是深入一点了解Spring配置文件结构搭配对于Bean装配的影响。

<!-- more -->

首先，我们设定一个简单的场景：播放器播放歌曲。所以基于此，我们定义两个接口：
```java
package demo;
// CD接口
public interface CompactDisc {
    void play();
}
```
```java
package demo;
// 媒体播放器接口
public interface MediaPlayer {
    void play();
}
```
按照是实际来讲，我们定义一个BlankDisc，空白的唱片，其包含三个属性：title、artist和tracks，分别代表了唱片的标题、歌手以及歌曲目录：
```java
package demo.cd;

import demo.CompactDisc;
import java.util.List;

public class BlankDisc implements CompactDisc {
    private String title;
    private String artist;
    private List<String> tracks; // 简化结构，只存放歌曲目录名称并保存为List
    public BlankDisc(String title, String artist, List<String> tracks) {
        this.title = title;
        this.artist = artist;
        this.tracks = tracks;
    }
    @Override
    public void play() {
        System.out.println("Playing " + title + " \n\tby " + artist);
        tracks.stream().forEach(t -> System.out.println(" \t>>> " + t));
    }
}
```
同样的，实现MediaPlayer接口，定义实际的唱片播放器，能够持有cd的引用，同时，这里我们并没有通过构造器来定义，原因是唱片播放器并非一定放有cd（当然代码没有对null进行约束，这是不好的，实际编写请勿这样编写）：
```java
package demo.player;

import demo.CompactDisc;
import demo.MediaPlayer;

public class CDPlayer implements MediaPlayer {
    private CompactDisc cd;

    public void setCd(CompactDisc cd) {
            this.cd = cd;
    }
    @Override
    public void play() {
        System.out.println("CDPlayer 开始播放: ");
        cd.play();
    }
}
```
接下来，要说明的是，Spring支持xml与Java文件同时存在的配置方式，这里我们也会这么做，尽可能的复杂化配置依赖，因为本片文章就是探讨各种配置文件交叉依赖的情形，并理清依赖的思路。

首先我们将CD类Bean与CDPlayer类Bean分离开来。

#### 首先是CD类Bean

##### Java类型配置文件

```java
package demo.config;

import demo.cd.BlankDisc;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.List;

@Configuration
public class CDConfig {
    @Bean
    public BlankDisc yeHuiMei() {
        List<String> tracks = new ArrayList<>();
        tracks.add("以父之名");
        tracks.add("懦夫");
        tracks.add("晴天");
        tracks.add("...");
        return new BlankDisc("YeHuiMei", "JayChou", tracks);
    }
}
```
在这个配置文件中，只定义了一个Bean，Bean id名称为yeHuiMei（方法名），同时也将相关的属性设置完毕。

##### xml类型配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="onTheRun"
          class="demo.cd.BlankDisc">
        <constructor-arg name="title" value="On The Run"/>
        <constructor-arg name="artist" value="JayChou"/>
        <constructor-arg name="tracks">
            <list>
                <value>牛仔很忙</value>
                <value>彩虹</value>
                <value>青花瓷</value>
                <value>...</value>
            </list>
        </constructor-arg>
    </bean>
</beans>
```
在这个xml配置文件中，我定义了一个名为onTheRun的Bean，同时也设置了对应的属性。

#### CDPlayer的Bean

##### Java类型配置文件

```java
@Configuration
public class CDPlayerConfig {
    @Bean
    public CDPlayer cdPlayerInJava(@Qualifier("onTheRun") CompactDisc cd) {
        CDPlayer cdPlayer = new CDPlayer();
        cdPlayer.setCd(cd);
        return cdPlayer;
    }
}
```
##### xml类型配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="cdPlayerInXML" class="demo.player.CDPlayer">
        <property name="cd" ref="yeHuiMei"/>
    </bean>
</beans>
```
目前配置文件搞定了，并且我们现在的配置以来结构如下：
![dependency](https://static-res.zhen.wang/images/post/2018-07-02-Spring/dependency.png)

当然，目前还是有一定的问题的，因为很显然，我们的配置文件都独立与彼此。尽管在CDPlayer中的配置文件通过相关的语法制定了CD Bean的选择（@Qualifier和ref），但是我们可以看到文件本身并没有明确的引入另外的配置文件，所以在IDEA中通常会有这样的提示：
![norefinjava](https://static-res.zhen.wang/images/post/2018-07-02-Spring/norefinjava.png)
![norefinxml](https://static-res.zhen.wang/images/post/2018-07-02-Spring/norefinxml.png)
同时打开，IDEA的项目结构Project Structs（win默认ctrl+shift+alt+s），点击左侧的Modules，可以看到Spring项目上右下角IDEA提示我们“Unmapped Spring configuration files”并列举除了上述的四个文件。

我们点击上面的+将所有的配置文件追踪上，刚刚所有的索引问题都OK了。此时，我们任意找到一个xml文件，可以看到左上方有一个小标志，点击并选择第一个：
![clickdpdiagram](https://static-res.zhen.wang/images/post/2018-07-02-Spring/clickdpdiagram.png)
打开之后就能够看到整个项目对于配置文件的依赖：
![dpdiagram](https://static-res.zhen.wang/images/post/2018-07-02-Spring/dpdiagram.png)
可以看到我们的项目（springdemo）具有一个是上下文应用模块，这个应用上下文包含了四份配置文件。但一定要注意，在后续我们加载配置文件的时候，必须要将有依赖关系的配置文件全部加载进来才能够读取到对应的Bean。这里我们进行一个简单的测试：
```java
@RunWith(SpringJUnit4ClassRunner.class)
// 设置所要加载的配置文件
@ContextConfiguration(locations = {"classpath:cdconfig.xml"})
public class CDPlayerTest {
    @Autowired
    @Qualifier("onTheRun")
    private CompactDisc cd;

    @Test
    public void cdShouldNotNull() {
        cd.play();
        assertNotNull(cd);
    }
}
```
这个测试是可以直接通过的，因为这里我们加载的是cdconfig.xml配置文件，里面我们定义了名为onTheRun的Bean，所以打印还有非空测试也通过：
![cdxmltest](https://static-res.zhen.wang/images/post/2018-07-02-Spring/cdxmltest.png)
然而接下来我们更换配置文件为cdplayerconfig.xml，相关注入如下：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:cdplayerconfig.xml"})
public class CDPlayerTest {
    @Autowired
    @Qualifier("cdPlayerInXML")
    private MediaPlayer mp;
    @Test
    public void mediaPlayerNotNull() {
        System.out.println(mp);
        mp.play();
        assertNotNull(mp);
    }
}
```
这里我们指定注入的就是xml中的CDPlayer Bean，然而，并不能通过测试，在错误提示中，我们可以找到这样一行：
```java
Cannot resolve reference to bean 'yeHuiMei' while setting bean property 'cd'; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'yeHuiMei' is defined
	at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveReference(BeanDefinitionValueResolver.java:328)
```
前面我们知道，cdPlayerInXML这个bean中我们还注入了Java配置文件下的名为yeHuiMei Bean，而在测试的过程中，我们只加载了cdplayerconfig.xml这个配置文件。所以实际上除了这个配置文件意外的其他bean都没有被Spring生成并放入Bean容器中。

也许会有疑问，在上面的Bean依赖图中，我们看到所有的配置文件都有已经被放入到了Spring Application Context中，为什么不被自动加载呢？道理很简单，这只是IDE的辅助而已。IDEA中的那个部分只是IDEA自身的一些辅助功能比如静态检查，所以需要我们手动的将这些文件给添加进去。当我们还是移除掉刚刚的结构之后，进行第一次的只对没有依赖的CDBean进行测试依然有效。

一定要明确，Spring的注入是发生在代码中的！不要被IDE遮蔽了双眼！这里何时会被注入呢？当我们配置了Spring的配置文件并将其加载进来了，当Spring遇到@Autowired等注入注解的时候，就会为我们注入Bean。

通常，当我们有多个配置文件的是，最优的结构思路是将多个配置文件导入到一个专门的独立的配置文件中，就像下面这样，我将开始的四个配置文件全部导入到一个名为AllConfig的Java配置文件：
```java
@Configuration
@Import({CDConfig.class, CDPlayerConfig.class})
// 一定要注意！！！classpath:后面一定不要带空格！否则会被识别为【[空格]cdconfig.xml】这样的文件名而不被找到，血的教训。
@ImportResource({"classpath:cdconfig.xml", "classpath:cdplayerconfig.xml"})
public class AllConfig {
}
```
然后在测试文件中我们将加载配置文件为Java配置文件AllConfig，此时，所有的以来问题全部解决：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {AllConfig.class})
public class CDPlayerTest {

    @Autowired
    @Qualifier("onTheRun")
    private CompactDisc cd;

    @Test
    public void cdShouldNotNull() {
        cd.play();
        assertNotNull(cd);
    }
    @Autowired
    @Qualifier("cdPlayerInXML")// 一开始由于配置文件没有引入全导致注入失败
    private MediaPlayer mp;
    @Test
    public void mediaPlayerNotNull() {
        System.out.println(mp);
        mp.play();
        assertNotNull(mp);
    }
}
```
![passed](https://static-res.zhen.wang/images/post/2018-07-02-Spring/passed.png)
