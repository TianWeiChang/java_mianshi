面试官：能说下 SpringBoot 启动原理吗？

`Spring Boot`为我们做的自动配置，确实方便快捷，但是对于新手来说，如果不大懂`Spring Boot`内部启动原理，以后难免会吃亏。所以这次博主就跟你们一起一步步揭开`Spring Boot`的神秘面纱，让它不在神秘。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwjuaK64mpNqkGKks0MZmkfPAN64fy4wHDQcztSKs3ruEv0MfVxGn1VQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们开发任何一个`Spring Boot`项目，都会用到如下的启动类

```java
@SpringBootApplication 
public class Application {     
     public static void main(String[] args) {        
     SpringApplication.run(Application.class, args);    
     }
}
```

从上面代码可以看出，`Annotation`定义（`@SpringBootApplication`）和类定义（`SpringApplication.run`）最为耀眼，所以要揭开`Spring Boot`的神秘面纱，我们要从这两位开始就可以了。

## 一、SpringBootApplication背后的秘密

@SpringBootApplication注解是Spring Boot的核心注解，它其实是一个组合注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
}
```

虽然定义使用了多个Annotation进行了原信息标注，但实际上重要的只有三个Annotation：

`@Configuration`（`@SpringBootConfiguration`点开查看发现里面还是应用了@Configuration）

`@EnableAutoConfiguration`

`@ComponentScan`

即 `@SpringBootApplication` = `(默认属性)@Configuration + @EnableAutoConfiguration + @ComponentScan`。

所以，如果我们使用如下的`Spring Boot`启动类，整个`Spring Boot`应用依然可以与之前的启动类功能对等：

```
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

每次写这3个比较累，所以写一个@SpringBootApplication方便点。接下来分别介绍这3个Annotation。

### 1、@Configuration

这里的@Configuration对我们来说不陌生，它就是JavaConfig形式的Spring Ioc容器的配置类使用的那个@Configuration，SpringBoot社区推荐使用基于JavaConfig的配置形式，所以，这里的启动类标注了@Configuration之后，本身其实也是一个IoC容器的配置类。

举几个简单例子回顾下，XML跟config配置方式的区别：

### （1）表达形式层面

基于XML配置的方式是这样：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
       default-lazy-init="true">
    <!--bean定义-->
</beans>
```

而基于JavaConfig的配置方式是这样：

```
 @Configuration 
 public class MockConfiguration{    
 //bean定义
 }
```

任何一个标注了`@Configuration`的Java类定义都是一个JavaConfig配置类。

### （2）注册bean定义层面

基于XML的配置形式是这样：

```
 <bean id="mockService" class="..MockServiceImpl">     ... </bean>
```

而基于JavaConfig的配置形式是这样的：

```
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl();
    }
}
```

任何一个标注了@Bean的方法，其返回值将作为一个bean定义注册到Spring的IoC容器，方法名将默认成该bean定义的id。

### （3）表达依赖注入关系层面

为了表达bean与bean之间的依赖关系，在XML形式中一般是这样：

```
 <bean id="mockService" class="..MockServiceImpl">     
     <propery name ="dependencyService" ref="dependencyService" /> 
 </bean>
 <bean id="dependencyService" class="DependencyServiceImpl"></bean>
```

而基于JavaConfig的配置形式是这样的：

```
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl(dependencyService());
    }

    @Bean
    public DependencyService dependencyService(){
        return new DependencyServiceImpl();
    }
}
```

如果一个bean的定义依赖其他bean，则直接调用对应的JavaConfig类中依赖bean的创建方法就可以了。

------

@Configuration：提到@Configuration就要提到他的搭档@Bean。使用这两个注解就可以创建一个简单的spring配置类，可以用来替代相应的xml配置文件。

```
<beans> 
     <bean id = "car" class="com.test.Car"> 
          <property name="wheel" ref = "wheel"></property> 
     </bean> 
     <bean id = "wheel" class="com.test.Wheel"></bean> 
  </beans>
```

相当于：

```
@Configuration
public class Conf {
    @Bean
    public Car car() {
        Car car = new Car();
        car.setWheel(wheel());
        return car;
    }

    @Bean
    public Wheel wheel() {
        return new Wheel();
    }
}
```

@Configuration的注解类标识这个类可以使用Spring IoC容器作为bean定义的来源。

@Bean注解告诉Spring，一个带有@Bean的注解方法将返回一个对象，该对象应该被注册为在Spring应用程序上下文中的bean。

### 2、@ComponentScan

@ComponentScan这个注解在Spring中很重要，它对应XML配置中的元素，@ComponentScan的功能其实就是自动扫描并加载符合条件的组件（比如@Component和@Repository等）或者bean定义，最终将这些bean定义加载到IoC容器中。

我们可以通过basePackages等属性来细粒度的定制@ComponentScan自动扫描的范围，如果不指定，则默认Spring框架实现会从声明@ComponentScan所在类的package进行扫描。

注：所以SpringBoot的启动类最好是放在root package下，因为默认不指定basePackages。

### 3、@EnableAutoConfiguration

个人感觉@EnableAutoConfiguration这个Annotation最为重要，所以放在最后来解读，大家是否还记得Spring框架提供的各种名字为@Enable开头的Annotation定义？比如@EnableScheduling、@EnableCaching、@EnableMBeanExport等，@EnableAutoConfiguration的理念和做事方式其实一脉相承，简单概括一下就是，**借助@Import的支持，收集和注册特定场景相关的bean定义。**

@EnableScheduling是通过@Import将Spring调度框架相关的bean定义都加载到IoC容器。

@EnableMBeanExport是通过@Import将JMX相关的bean定义加载到IoC容器。

而`@EnableAutoConfiguration`也是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器，仅此而已！

`@EnableAutoConfiguration`会根据类路径中的jar依赖为项目进行自动配置，如：添加了`spring-boot-starter-web`依赖，会自动添加Tomcat和Spring MVC的依赖，`Spring Boot`会对Tomcat和`Spring MVC`进行自动配置。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwxslykaocGlj0VNfmBZ0a9o16JNObx1yr8dvFticpqbiaDqnwN4FKlnuA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

@EnableAutoConfiguration作为一个复合`Annotation`，其自身定义关键信息如下：

```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

其中，最关键的要属`@Import(EnableAutoConfigurationImportSelector.class)`，借助`EnableAutoConfigurationImportSelector`，`@EnableAutoConfiguration`可以帮助`Spring Boot`应用将所有符合条件的@Configuration配置都加载到当前`Spring Boot`创建并使用的`IOC`容器。

就像一只“八爪鱼”一样，借助于Spring框架原有的一个工具类：`SpringFactoriesLoader`的支持，`@EnableAutoConfiguration`可以智能的自动配置功效才得以大功告成！

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwQHGic7oXY3TKAZ7xuDzyWZ6pDdia7ianXVpNWXJKhZNMHBJ6iaRI309ibsw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 自动配置幕后英雄：SpringFactoriesLoader详解

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从指定的配置文件META-INF/spring.factories加载配置。

```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key，获取对应的一组@Configuration类。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQw483dWY8ibuYTOvkFqy3o3x6Mjm6F7RDz6dWfhhGw3RlVibDtDO1Qeicaw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图就是从`Spring Boot`的`autoconfigure`依赖包中的`META-INF/spring.factories`配置文件中摘录的一段内容，可以很好地说明问题。

所以，`@EnableAutoConfiguration`自动配置的魔法骑士就变成了：**从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射（Java Refletion）实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。**

## 二、深入探索SpringApplication执行流程

`SpringApplication`的run方法的实现是我们本次旅程的主要线路，该方法的主要流程大体可以归纳如下：

1） 如果我们使用的是`SpringApplication`的静态run方法，那么，这个方法里面首先要创建一个SpringApplication对象实例，然后调用这个创建好的`SpringApplication`的实例方法。在SpringApplication实例初始化的时候，它会提前做几件事情：

根据classpath里面是否存在某个特征类

（`org.springframework.web.context.ConfigurableWebApplicationContext`）来决定是否应该创建一个为Web应用使用的`ApplicationContext`类型。

使用`SpringFactoriesLoader`在应用的classpath中查找并加载所有可用的`ApplicationContextInitializer`。

使用`SpringFactoriesLoader`在应用的classpath中查找并加载所有可用的`ApplicationListener`。

推断并设置main方法的定义类。

2） SpringApplication实例初始化完成并且完成设置后，就开始执行run方法的逻辑了，方法执行伊始，首先遍历执行所有通过SpringFactoriesLoader可以查找到并加载的SpringApplicationRunListener。调用它们的started()方法，告诉这些SpringApplicationRunListener，“嘿，SpringBoot应用要开始执行咯！”。

3） 创建并配置当前Spring Boot应用将要使用的Environment（包括配置要使用的PropertySource以及Profile）。

4） 遍历调用所有SpringApplicationRunListener的environmentPrepared()的方法，告诉他们：“当前SpringBoot应用使用的Environment准备好了咯！”。

5） 如果SpringApplication的showBanner属性被设置为true，则打印banner。

6） 根据用户是否明确设置了applicationContextClass类型以及初始化阶段的推断结果，决定该为当前SpringBoot应用创建什么类型的ApplicationContext并创建完成，然后根据条件决定是否添加ShutdownHook，决定是否使用自定义的BeanNameGenerator，决定是否使用自定义的ResourceLoader，当然，最重要的，将之前准备好的Environment设置给创建好的ApplicationContext使用。

7） ApplicationContext创建好之后，SpringApplication会再次借助Spring-FactoriesLoader，查找并加载classpath中所有可用的ApplicationContext-Initializer，然后遍历调用这些ApplicationContextInitializer的`initialize（applicationContext）`方法来对已经创建好的`ApplicationContext`进行进一步的处理。

8） 遍历调用所有`SpringApplicationRunListener`的`contextPrepared()`方法。

9） 最核心的一步，将之前通过`@EnableAutoConfiguration`获取的所有配置以及其他形式的IoC容器配置加载到已经准备完毕的`ApplicationContext`。

10） 遍历调用所有`SpringApplicationRunListener`的`contextLoaded()`方法。

11） 调用`ApplicationContext`的refresh()方法，完成IoC容器可用的最后一道工序。

12） 查找当前`ApplicationContext`中是否注册有`CommandLineRunner`，如果有，则遍历执行它们。

13） 正常情况下，遍历执行`SpringApplicationRunListener`的finished()方法、（如果整个过程出现异常，则依然调用所有SpringApplicationRunListener的finished()方法，只不过这种情况下会将异常信息一并传入处理）

### 去除事件通知点后，整个流程如下：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwG4o5BydwF6VXShtcrLABlTYS12dhjNxPgPNldGosSH7icmoudBoCIgw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

------

本文以调试一个实际的`Spring Boot`启动程序为例，参考流程中主要类类图，来分析其启动逻辑和自动化配置原理。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQw8q4mUHuncicKDhvj0QVSooqTPqaibbSuibX5VGLnIlpcJB39HCze7dHcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 总览

上图为`Spring Boot`启动结构图，我们发现启动流程主要分为三个部分，第一部分进行`SpringApplication`的初始化模块，配置一些基本的环境变量、资源、构造器、监听器，第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块，第三部分是自动化配置模块，该模块作为`Spring Boot`自动配置核心，在后面的分析中会详细讨论。在下面的启动程序中我们会串联起结构中的主要功能。

### 启动

每个`Spring Boot`程序都有一个主入口，也就是main方法，main里面调用`SpringApplication.run()`启动整个spring-boot程序，该方法所在类需要使用`@SpringBootApplication`注解，以及`@ImportResource`注解(if need)，`@SpringBootApplication`包括三个注解，功能如下：

`@EnableAutoConfiguration`：`Spring Boot`根据应用所声明的依赖来对Spring框架进行自动配置。

`@SpringBootConfiguration(内部为@Configuration)`：被标注的类等于在spring的XML配置文件中(`applicationContext.xml`)，装配所有bean事务，提供了一个spring的上下文环境。

`@ComponentScan`：组件扫描，可自动发现和装配Bean，默认扫描`SpringApplication`的run方法里的`Booter.class`所在的包路径下文件，所以最好将该启动类放到根包路径下。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwfALMtia3fWq4l1Ecq2ytSXIkkTOiaLStGnwdbTBrfb0f28nve0qSRlsQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### SpringBoot启动类

首先进入run方法

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwfXlWiawA9qJ15sgHrtTa5oKHicibmdv90kyo6qa14aIiaMne4sEq89hkDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

run方法中去创建了一个SpringApplication实例，在该构造方法内，我们可以发现其调用了一个初始化的initialize方法

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwGQLMgKgTzLX1v0oeQM83UBgcpg5foy1p5MfTDxMsuPCGB1J3xicUQZg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQw3Tpsd7CUKiaFctKuT9gMNm5WzawZxB49HOdcq6Oic82UyoCu9ZFhp5kA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里主要是为`SpringApplication`对象赋一些初值。构造函数执行完毕后，我们回到run方法

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwatMvgeIhHCGDkuibgF5f2bnRMX2bV8d06RsqFwHmd1v7yuvKAOJaJAw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 该方法中实现了如下几个关键步骤：

1、创建了应用的监听器`SpringApplicationRunListeners`并开始监听

2、加载`Spring Boot`配置环境(`ConfigurableEnvironment`)，如果是通过web容器发布，会加载`StandardEnvironment`，其最终也是继承了`ConfigurableEnvironment`，类图如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwOjgeGQSIbGyqUVXWS2nXgn3sH7jyMmNF0zJ33hwah1TSZDEwSp1hcA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看出，*Environment最终都实现了PropertyResolver接口，我们平时通过environment对象获取配置文件中指定Key对应的value方法时，就是调用了propertyResolver接口的getProperty方法

3、配置环境(Environment)加入到监听器对象中(SpringApplicationRunListeners)

4、创建run方法的返回对象：ConfigurableApplicationContext(应用配置上下文)，我们可以看一下创建方法：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwV5cqgkJUReWUQbuuj56jAibB38clqDhFm4JE2nFa34vCK9Kdtcb0QJg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

方法会先获取显式设置的应用上下文(applicationContextClass)，如果不存在，再加载默认的环境配置（通过是否是web environment判断），默认选择AnnotationConfigApplicationContext注解上下文（通过扫描所有注解类来加载bean），最后通过BeanUtils实例化上下文对象，并返回。

ConfigurableApplicationContext类图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwJJKjdViadwfeMtfgzqx9bSUpwPB9hLHtx0b6aYkZ0cLvbnUEMpXkkBA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 主要看其继承的两个方向：

LifeCycle：生命周期类，定义了start启动、stop结束、isRunning是否运行中等生命周期空值方法

ApplicationContext：应用上下文类，其主要继承了beanFactory(bean的工厂类)

5、回到run方法内，prepareContext方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联

6、接下来的refreshContext(context)方法(初始化方法如下)将是实现spring-boot-starter-*(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwiakYrvuStAIaBAoibYOM3FsFOyzr4BDdfPic3Z5NKicKorEC5p8nibdWxyg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

配置结束后，Springboot做了一些基本的收尾工作，返回了应用环境上下文。回顾整体流程，Springboot的启动，主要创建了配置环境(environment)、事件监听(listeners)、应用上下文(applicationContext)，并基于以上条件，在容器中开始实例化我们需要的Bean，至此，通过SpringBoot启动的程序已经构造完成，接下来我们来探讨自动化配置是如何实现。

------

### 自动化配置

之前的启动结构图中，我们注意到无论是应用初始化还是具体的执行过程，都调用了SpringBoot自动配置模块。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwPb5wZpzicaN0ceCJN8vWWxD7VUsNLSaHEcRQHwdZrc3kbmsxHTnjHVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### SpringBoot自动配置模块

该配置模块的主要使用到了SpringFactoriesLoader，即Spring工厂加载器，该对象提供了loadFactoryNames方法，入参为factoryClass和classLoader，即需要传入上图中的工厂类名称和对应的类加载器，方法会根据指定的classLoader，加载该类加器搜索路径下的指定文件，即spring.factories文件，传入的工厂类为接口，而文件中对应的类则是接口的实现类，或最终作为实现类，所以文件中一般为如下图这种一对多的类名集合，获取到这些实现类的类名后，loadFactoryNames方法返回类名集合，方法调用方得到这些集合后，再通过反射获取这些类的类对象、构造方法，最终生成实例。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwjrOe5kdxZyejqOiaAK4hcSykGics6l83dzuII96yTfVeUjMOUoQibSlicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 工厂接口与其若干实现类接口名称

下图有助于我们形象理解自动配置流程。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwXmN0WGhcmtK9161BkUxTIYaZibiaxXFG0fOco1mMPRpibLHlWbTuTWTQw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### SpringBoot自动化配置关键组件关系图

mybatis-spring-boot-starter、spring-boot-starter-web等组件的META-INF文件下均含有spring.factories文件，自动配置模块中，SpringFactoriesLoader收集到文件中的类全名并返回一个类全名的数组，返回的类全名通过反射被实例化，就形成了具体的工厂实例，工厂实例来生成组件具体需要的bean。

### 之前我们提到了EnableAutoConfiguration注解，其类图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwvG0ZWGiaBYFpWf6YyCbw7kG7XCcQwKvP0ibQZmT0sKzEFKCqwXKh04DA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以发现其最终实现了ImportSelector(选择器)和BeanClassLoaderAware(bean类加载器中间件)，重点关注一下AutoConfigurationImportSelector的selectImports方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwcSoFSkBTFmQQ4bzUZZw2DaOBNgpP2EaCuGiat3pxFouf82W0ExImwfw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

该方法在springboot启动流程——bean实例化前被执行，返回要实例化的类信息列表。我们知道，如果获取到类信息，spring自然可以通过类加载器将类加载到jvm中，现在我们已经通过spring-boot的starter依赖方式依赖了我们需要的组件，那么这些组建的类信息在select方法中也是可以被获取到的，不要急我们继续向下分析。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwAcT9nTrKFDm7GKIZfIAagp7oTUsEiapG0zlPic20Mic6SNSZ0qnhcPqbg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

该方法中的getCandidateConfigurations方法，通过方法注释了解到，其返回一个自动配置类的类名列表，方法调用了loadFactoryNames方法，查看该方法

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwDRmraAaawic54Hpm4vuAibiao0CsniaDibwCzcmjhQEt3adwuPw8v932xzw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在上面的代码可以看到自动配置器会根据传入的factoryClass.getName()到项目系统路径下所有的spring.factories文件中找到相应的key，从而加载里面的类。我们就选取这个mybatis-spring-boot-autoconfigure下的spring.factories文件

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQw2gryoesSApP3QyYVqqlLjBjf0rY08LpYdAHzzEmBUpjVFPDDXxJOqA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

进入org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration中，主要看一下类头：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwtueREPyV1wrStfnCozEujV3uvwrVf41vCQNicUMp8v5aS5DY8K1XtVA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

发现Spring的`@Configuration`，俨然是一个通过注解标注的springBean，继续向下看，

`@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class})`这个注解的意思是：当存在`SqlSessionFactory.class, SqlSessionFactoryBean.class`这两个类时才解析MybatisAutoConfiguration配置类，否则不解析这一个配置类，make sence，我们需要mybatis为我们返回会话对象，就必须有会话工厂相关类。

`@CondtionalOnBean(DataSource.class)：`只有处理已经被声明为bean的dataSource。

`@ConditionalOnMissingBean(MapperFactoryBean.class)`这个注解的意思是如果容器中不存在name指定的bean则创建bean注入，否则不执行（该类源码较长，篇幅限制不全粘贴）

以上配置可以保证sqlSessionFactory、sqlSessionTemplate、dataSource等mybatis所需的组件均可被自动配置，@Configuration注解已经提供了Spring的上下文环境，所以以上组件的配置方式与Spring启动时通过mybatis.xml文件进行配置起到一个效果。通过分析我们可以发现，只要一个基于SpringBoot项目的类路径下存在SqlSessionFactory.class, SqlSessionFactoryBean.class，并且容器中已经注册了dataSourceBean，就可以触发自动化配置，意思说我们只要在maven的项目中加入了mybatis所需要的若干依赖，就可以触发自动配置，但引入mybatis原生依赖的话，每集成一个功能都要去修改其自动化配置类，那就得不到开箱即用的效果了。所以Spring-boot为我们提供了统一的starter可以直接配置好相关的类，触发自动配置所需的依赖(mybatis)如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwMRMotqgnkMmxV7poMJhTdt3LMY2eBYuXkd4dmRWL8ThQJaMrNCbW9g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里是截取的mybatis-spring-boot-starter的源码中pom.xml文件中所有依赖：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tO7NEN7wjr4UKrNJCKxj9L0u98yWCIQwDWib2tPC94YJWuudtDmiczlergYRbAPAibG7MvEN1BnQAnNNK718DY6YA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

因为maven依赖的传递性，我们只要依赖starter就可以依赖到所有需要自动配置的类，实现开箱即用的功能。也体现出`Spring boot`简化了Spring框架带来的大量XML配置以及复杂的依赖管理，让开发人员可以更加关注业务逻辑的开发。

