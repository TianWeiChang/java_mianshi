面试官：说说你对Spring中的@Bean注解的理解

随着SpringBoot的流行，我们现在更多采用基于注解式的配置从而替换掉了基于XML的配置，所以本篇文章我们主要探讨基于注解的@Bean以及和其他注解的使用； 

## 基础概念

1. @Bean：`Spring`的`@Bean`注解用于告诉方法，产生一个Bean对象，然后这个Bean对象交给Spring管理。产生这个Bean对象的方法Spring只会调用一次，随后这个Spring将会将这个Bean对象放在自己的IOC容器中；
2. `Spring IOC` 容器管理一个或者多个bean，这些bean都需要在`@Configuration`注解下进行创建，在一个方法上使用@Bean注解就表明这个方法需要交给Spring进行管理；
3. @Bean是一个方法级别上的注解，主要用在`@Configuration`注解的类里，也可以用在`@Component`注解的类里。添加的bean的id为方法名；
4. 使用Bean时，即是把已经在xml文件中配置好的Bean拿来用，完成属性、方法的组装；比如@Autowired , @Resource，可以通过`byTYPE（@Autowired）`、`byNAME（@Resource）`的方式获取Bean；
5. 注册Bean时，`@Component` ,` @Repository `,` @ Controller` ,` @Service `,` @Configration`这些注解都是把你要实例化的对象转化成一个Bean，放在IoC容器中，等你要用的时候，它会和上面的`@Autowired` ,` @Resource`配合到一起，把对象、属性、方法完美组装；
6. `@Configuration`与@Bean结合使用：`@Configuration`可理解为用spring的时候xml里面的标签，@Bean可理解为用`spring`的时候`xml`里面的标签；

## 案例

快速搭建一个maven项目并配置好所需要的Spring 依赖 。

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>4.3.13.RELEASE</version>
</dependency>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-beans</artifactId>
  <version>4.3.13.RELEASE</version>
</dependency>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-core</artifactId>
  <version>4.3.13.RELEASE</version>
</dependency>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-web</artifactId>
  <version>4.3.13.RELEASE</version>
</dependency>
```

在src根目录下创建一个`AppConfig`的配置类，这个配置类也就是管理一个或多个bean 的配置类，并在其内部声明一个`myBean`的bean，并创建其对应的实体类

```java
@Configuration
public class AppConfig {

    // 使用@Bean 注解表明myBean需要交给Spring进行管理
    // 未指定bean 的名称，默认采用的是 "方法名" + "首字母小写"的配置方式
    @Bean
    public MyBean myBean(){
        return new MyBean();
    }
}

public class MyBean {

    public MyBean(){
        System.out.println("MyBean Initializing");
    }
}
```

然后再创建一个测试类`SpringBeanApplicationTests`，测试上述代码的正确性 。

```java
public class SpringBeanApplicationTests {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        context.getBean("myBean");
    }
}
```

输出结果：

> MyBean Initializing 

## 源码分析

深入了解@Bean注解的源代码。

```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {
    @AliasFor("name")
    String[] value() default {};
    @AliasFor("value")
    String[] name() default {};
    Autowire autowire() default Autowire.NO;
    String initMethod() default "";
    String destroyMethod() default AbstractBeanDefinition.INFER_METHOD;
}
```

## @Bean的属性

1. value：bean别名和name是相互依赖关联的，value,name如果都使用的话值必须要一致；
2. name：bean名称，如果不写会默认为注解的方法名称；
3. autowire：自定装配默认是不开启的，建议尽量不要开启，因为自动装配不能装配基本数据类型、字符串、数组等，这是自动装配设计的局限性，并且自动装配不如依赖注入精确；
4. initMethod：bean的初始化之前的执行方法，该参数一般不怎么用，因为完全可以在代码中实现；
5. destroyMethod：默认使用javaConfig配置的bean，如果存在close或者shutdown方法，则在bean销毁时会自动执行该方法，如果你不想执行该方法，则添加@Bean(destroyMethod="")来防止出发销毁方法；
6. 如果发现销毁方法没有执行，原因是bean销魂之前程序已经结束了，可以手动close下如下：

```java
AnnotationConfigApplicationContext applicationContext2 = new AnnotationConfigApplicationContext(MainConfig.class);
        User bean2 = applicationContext2.getBean(User.class);
        System.out.println(bean2);
        //手动执行close方法
        applicationContext2.close();
```

运行结果：

> 初始化用户bean之前执行
>
> User [userName=张三, age=26] 
>
> bean销毁之后执行 

我们发现@baen注解的@Target是ElementType.METHOD,ElementType.ANNOTATION_TYPE也就说@Bean注解可以在使用在方法上，以及一个注释类型声明

## @Bean 注解与其他注解一起使用

@Bean注解不止这几个属性，它还能和其他的注解一起配合使用，请继续往下看：

@Profile 注解
@Profile的作用是把一些meta-data进行分类，分成Active和InActive这两种状态，然后你可以选择在active 和在Inactive这两种状态下配置bean，在Inactive状态通常的注解有一个！操作符，通常写为：@Profile("!p"),这里的p是Profile的名字。

三种设置方式：

- 可以通过ConfigurableEnvironment.setActiveProfiles()以编程的方式激活。
- 可以通过AbstractEnvironment.ACTIVE_PROFILES_PROPERTY_NAME (spring.profiles.active )属性设置为JVM属性。
- 作为环境变量，或作为web.xml 应用程序的Servlet 上下文参数。也可以通过@ActiveProfiles 注解在集成测试中以声明方式激活配置文件。

## 作用域

- 作为类级别的注解在任意类或者直接与@Component 进行关联，包括@Configuration 类
- 作为原注解，可以自定义注解
- 作为方法的注解作用在任何方法

注意：

>  如果一个配置类使用了Profile 标签或者@Profile 作用在任何类中都必须进行启用才会生效，如果@Profile({“p1”,"!p2"}) 标识两个属性，那么p1 是启用状态 而p2 是非启用状态的。

例如：

```java
@Profile("dev")
public @Bean("activityMongoFactory")
    MongoDbFactory activityMongoFactoryDev(MongoClient activityMongo) {
        return new SimpleMongoDbFactory(activityMongo, stringValueResolver.resolveStringValue("${mongodb.dev.database}"));
}
```

## @Scope 注解

在Spring中对于bean的默认处理都是单例的，我们通过上下文容器.getBean方法拿到bean容器，并对其进行实例化，这个实例化的过程其实只进行一次，即多次getBean 获取的对象都是同一个对象，也就相当于这个bean的实例在IOC容器中是public的，对于所有的bean请求来讲都可以共享此bean。

- 如果我们不想把这个bean被所有的请求共享或者说每次调用我都想让它生成一个新的bean实例，该怎么实现呢？

bean的多个实例

bean的非单例原型范围会使每次发出对该特定bean的请求时都创建新的bean实例，也就是说，bean被注入另一个bean，或者通过对容器的getBean()方法调用来请求它。

下面，我们通过一个示例来说明bean的多个实例 。

新建一个`ConfigScope`配置类，用来定义多例的bean ：

```java
@Configuration
public class ConfigScope {

    /**
     * 为myBean起两个名字，b1 和 b2
     * @Scope 默认为 singleton，但是可以指定其作用域
     * prototype 是多例的，即每一次调用都会生成一个新的实例。
     */
    @Bean({"b1","b2"})
    @Scope("prototype")
    public MyBean myBean(){
        return new MyBean();
    }
}
```

注意：prototype代表bean对象的定义为任意数量的对象实例，所以指定为prototype属性可以定义为多例，也会每一次调用都会生成一个新的实例。

| Scope       | 详解                                                         |
| ----------- | ------------------------------------------------------------ |
| singleton   | 默认单例的bean定义信息，对于每个IOC容器来说都是单例对象      |
| prototype   | bean对象的定义为任意数量的对象实例                           |
| request     | bean对象的定义为一次HTTP请求的生命周期，也就是说，每个HTTP请求都有自己的bean实例，它是在单个bean定义的后面创建的。仅仅在web-aware的上下文中有效 |
| session     | bean对象的定义为一次HTTP会话的生命周期。仅仅在web-aware的上下文中有效 |
| application | bean对象的定义范围在ServletContext生命周期内。仅仅在web-aware的上下文中有效 |
| websocket   | bean对象的定义为WebSocket的生命周期内。仅仅在web-aware的上下文中有效 |

> singleton和prototype 一般都用在普通的Java项目中，而request、session、application、websocket都用于web应用中。

## @Lazy 注解

表明一个bean 是否延迟加载，可以作用在方法上，表示这个方法被延迟加载；可以作用在@Component (或者由@Component 作为原注解) 注释的类上，表明这个类中所有的bean 都被延迟加载。如果没有@Lazy注释，或者@Lazy 被设置为false，那么该bean 就会急切渴望被加载；除了上面两种作用域，@Lazy 还可以作用在@Autowired和@Inject注释的属性上，在这种情况下，它将为该字段创建一个惰性代理，作为使用ObjectFactory或Provider的默认方法。 

下面来演示一下： 

```java
@Lazy
@Configuration
@ComponentScan(basePackages = "com.spring.configuration.pojo")
public class AppConfigWithLazy {

    @Bean
    public MyBean myBean(){
        System.out.println("myBean Initialized");
        return new MyBean();
    }

    @Bean
    public MyBean IfLazyInit(){
        System.out.println("initialized");
        return new MyBean();
    }
}
```

## @Primary 注解

指示当多个候选者有资格自动装配依赖项时，应优先考虑bean。此注解在语义上就等同于在Spring XML中定义的bean 元素的primary属性。注意：除非使用component-scanning进行组件扫描，否则在类级别上使用@Primary不会有作用。如果@Primary 注解定义在XML中，那么@Primary 的注解元注解就会忽略，相反的话就可以优先使用。

@Primary 的两种使用方式：

- 与@Bean 一起使用，定义在方法上，方法级别的注解
- 与@Component 一起使用，定义在类上，类级别的注解

新建一个AppConfigWithPrimary类，在方法级别上定义@Primary注解

```java
@Configuration
public class AppConfigWithPrimary {

    @Bean
    public MyBean myBeanOne(){
        return new MyBean();
    }

    @Bean
    @Primary
    public MyBean myBeanTwo(){
        return new MyBean();
    }
}
```

上面代码定义了两个bean ，其中myBeanTwo 由@Primary 进行标注，表示它首先会进行注册，使用测试类进行测试 。





