---

title: 极简SpringBoot指南-Chapter05-SpringBoot中的AOP面向切面编程简介
date: 2021-08-10
tags:
 - Java
 - SpringBoot

---

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)

# Chapter05-SpringBoot中的AOP面向切面编程简介

<!-- more -->

在上一章中，我们编写了一款基于SpringBoot的书籍信息管理Web应用，实现了对书籍信息的增删查改操作。现在，我们有了一个新的需求：为了方便后台服务监管我们的Web服务的请求耗时，我们需要增强一下对我们的Web应用，希望对每个请求都能够打印一下处理耗时。

## 基本方式

为了实现这样的需求，我们首先以**获取指定ID的书籍信息**这个API为例，开始进行编程：

```java
    @GetMapping("{id}")
    public Book getBookById(@PathVariable("id") String id) {
        long currentTimeMillis = System.currentTimeMillis();
        try {
            // 为了更加明显，我模拟了一个耗时
            Thread.sleep(500); 
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Optional<Book> first = this.bookList
                .stream()
                .filter(b -> b.getId().equals(id))
                .findFirst();
        System.out.printf("处理耗时：%d ms %n", (System.currentTimeMillis() - currentTimeMillis));
        return first.orElse(null);
    }
```

为了模拟一个耗时，我在请求处理的时候对当前处理线程sleep500ms。然后使用对应postman进行请求调用，调用后查看控制台打印：

```
处理耗时：513 ms 
```

效果还行，但是现在需要对所有的调用都进行日志记录呢？有的同学可能会说，直接开写，一个一个加。牛！此外，我今天的需求是耗时打印，我以后可能需要耗时上传进行预警了，再后来我还希望统计各个API的调用接口，似乎我们愈来愈无法控制这些需求了。每变更一个需求，都需要我们去对每个API进行修改。

还好，我们还有一大杀器：AOP。

## AOP

AOP全称`Aspect Oriented Programming`意为面向切面编程，也叫做面向方法编程，是通过预编译方式和运行期动态代理的方式实现不修改源代码的情况下给程序动态统一添加功能的技术。

```
流程起点
 |
 |
---> 切入点1
 |
 |
---> 切入点2
 |
 V
流程终点
```

上图的流程中，我们可以在任何希望的时候**切入**处理。其好处的就是是的业务逻辑各个部分之间的耦合度降低，提高程序的可重用性。我们可以在完全不侵入业务逻辑代码的情况下就完成各个阶段的切入处理。

### 核心术语

#### 连接点（JoinPoint）

连接点是在应用执行过程中能够插入切面（Aspect）的一个点。这些点可以是调用方法时、甚至修改一个字段时。它是一个虚拟的概念，例如坐地铁的时候，每一个站都可以下车，那么这每一个站都是一个连接点。假如一个对象中有多个方法，那么这个每一个方法就是一个连接点。

#### 切入点（Pointcut）

切入点是一些特殊的连接点，是具体附加通知的地方。例如坐地铁的时候，具体在某个站下车，那这个站就是切入点。

#### 通知（Advice）

在某个特定的Pointcut切点上需要的执行的动作，如日志记录，权限校验等具体要应用切入点的代码。

五种通知类型：

- 环绕通知：**@Around**
- 前置通知：**@Before**
- 返回通知：**@After**
- 正常返回通知：**@AfterReturning**
- 异常返回通知：**@AfterThrowing**

#### 切面（Aspect）

切面是**通知**和**切入点**的结合，通知规定了在什么时机干什么事，切入点规定了在什么地方。如“在8点钟在天府广场站下车“ 就是一个切面：时间8点，动作下车就是一个通知；西站就是一个切入点。

对于概念术语还是很抽象，我们直接编写一个切面吧。编写切面之前，首先需要引入相关的依赖。因为切面相关的模块是可选模块，我们在pom中添加如下的依赖：

```xml
<dependencies>
	<!-- 其他依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
	</dependency>
</dependencies>
```

完成依赖导入后，我们编写一个切面类：

```java
/**
 * 定义一个日志切面
 */
@Aspect
@Component // 切面不是Bean，需要添加注解
public class LogAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(LogAspect.class);

    /**
     * 定义切点表明要通知的地方
     * 这里使用了pointcut expression表达式，具体语法请自行搜索
     * 这里解释为包 com.compilemind.guide包以及子包的所有类的所有公共方法
     */
    @Pointcut("execution(* com.compilemind.guide..*.*(..))")
    public void webLog() {
    }

    /**
     * 指代上面的切点：webLog，并且是调用前执行
     */
    @Before("webLog()")
    public void doBefore(JoinPoint joinPoint) {
        ServletRequestAttributes requestAttributes =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (requestAttributes == null) {
            return;
        }
        HttpServletRequest request = requestAttributes.getRequest();
        LOGGER.info("发生请求：" + request.getRequestURI());
        LOGGER.info("调用方法：" + joinPoint.getSignature());
    }

    /**
     * 一个关于对应切点的环绕执行的处理
     */
    @Around("webLog()")
    public Object around(ProceedingJoinPoint joinPoint) {
        long currentTimeMillis = System.currentTimeMillis();
        // 调用参数
        Object[] invokeArgs = joinPoint.getArgs();
        // 返回数据
        Object returnObj;
        try {
            // 执行对应的方法，得到结果
            returnObj = joinPoint.proceed(invokeArgs);
        } catch (Throwable e) {
            LOGGER.error("统计某方法执行耗时环绕通知出错", e);
            return null;
        }
        LOGGER.info("处理耗时：{} ms", System.currentTimeMillis() - currentTimeMillis);
        return returnObj;
    }
}
```

这个切面由以下几个部分组成：

1. 在类上使用`@Aspect`注解标记为切面，使用`@Component`注解标记为组件，由Spring管理；
2. 编写方法`webLog`，并在其方法上添加注解`@Pointcut`，并按照规则填写切点的位置；
3. 分别编写由`@Before`和`@Around`注解标记的方法，用以处理对应的切点位置**处理前**和**整个环绕**的处理代码。

最后，让我们再次启动程序，进行相关的API调用，可以看到输出：

```
2021-08-09 16:42:46.494  INFO 18528 --- [nio-8080-exec-2] c.c.guide.chapter04_05.aspect.LogAspect  : 发生请求：/books/1
2021-08-09 16:42:46.495  INFO 18528 --- [nio-8080-exec-2] c.c.guide.chapter04_05.aspect.LogAspect  : 调用方法：Book com.compilemind.guide.chapter04_05.controller.BookController.getBookById(String)
处理耗时：502 ms 
2021-08-09 16:42:47.004  INFO 18528 --- [nio-8080-exec-2] c.c.guide.chapter04_05.aspect.LogAspect  : 处理耗时：510 ms
```

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)

