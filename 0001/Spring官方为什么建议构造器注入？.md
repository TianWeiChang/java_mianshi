## Spring官方为什么建议构造器注入？

来源：https://juejin.cn/post/6844904056230690824

## **前言** 

本章的内容主要是想探讨我们在进行 Spring 开发过程当中，关于依赖注入的几个知识点。感兴趣的读者可以先看下以下问题：

- `@Autowired`，`@Resource`，`@Inject` 三个注解的区别
- 当你在使用`@Autowired`时，是否有出现过`Field injection is not recommended`的警告？你知道这是为什么吗？
- Spring 依赖注入有哪几种方式？官方是怎么建议使用的呢？

如果你对上述问题都了解，那我个人觉得你的开发经验应该是不错的👍。

下面我们就依次对上述问题进行解答，并且总结知识点。

## 三个注解的区别

Spring 支持使用`@Autowired`, `@Resource`,  `@Inject` 三个注解进行依赖注入。

下面来介绍一下这三个注解有什么区别。

### **@Autowired**

`@Autowired`为Spring 框架提供的注解，需要导入包`org.springframework.beans.factory.annotation.Autowired`。

这里先给出一个示例代码，方便讲解说明：

```java
public interface Svc {
    void sayHello();
}
@Service
public class SvcA implements Svc {

    @Override
    public void sayHello() {
        System.out.println("hello, this is service A");
    }

}
@Service
public class SvcB implements Svc {
    @Override
    public void sayHello() {
        System.out.println("hello, this is service B");
    }

}
@Service
public class SvcC implements Svc {

    @Override
    public void sayHello() {
        System.out.println("hello, this is service C");
    }
}
```

测试类：

```java
@SpringBootTest
public class SimpleTest {

    @Autowired
    // @Qualifier("svcA")
    Svc svc;

    @Test
    void rc() {
        Assertions.assertNotNull(svc);
        svc.sayHello();
    }

}
```

装配顺序：

1. 按照`type`在上下文中查找匹配的bean

   查找type为Svc的bean

2. 如果有多个bean，则按照`name`进行匹配

3. 1. 如果有`@Qualifier`注解，则按照`@Qualifier`指定的`name`进行匹配

      查找name为svcA的bean

   2. 如果没有，则按照变量名进行匹配

      查找name为svc的bean

4. 匹配不到，则报错。（`@Autowired(required=false)`，如果设置`required`为`false`(默认为`true`)，则注入失败时不会抛出异常）

### **@Inject**

在 Spring 的环境下，`@Inject`和`@Autowired` 是相同的，因为它们的依赖注入都是使用`AutowiredAnnotationBeanPostProcessor`来处理的。

![图片](https://mmbiz.qpic.cn/mmbiz/6fuT3emWI5KUJgwXhns2cguibibE2IJrbZ9okficTuO8ys88SiaBURfYSf2LdNZsrvxPlmYsQpmcN2iblOofpYV9j6w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`@Inject`是 JSR-330 定义的规范，如果使用这种方式，切换到`Guice`也是可以的。

> Guice 是 google 开源的轻量级 DI 框架

如果硬要说两个的区别，首先`@Inject`是 Java EE 包里的，在 SE 环境需要单独引入。另一个区别在于`@Autowired`可以设置`required=false`而`@Inject`并没有这个属性。

### **@Resource**

`@Resource`是 JSR-250 定义的注解。Spring 在 `CommonAnnotationBeanPostProcessor`实现了对`JSR-250`的注解的处理，其中就包括`@Resource`。

![图片](https://mmbiz.qpic.cn/mmbiz/6fuT3emWI5KUJgwXhns2cguibibE2IJrbZxebL864D7iafEdG5hMyP20sK5Nibic2DczZpcSMM7TW5yw1BO5r90dz7w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`@Resource`有两个重要的属性：`name`和`type`，而Spring 将`@Resource`注解的`name`属性解析为bean的名字，而`type`属性则解析为bean的类型。

装配顺序：

1. 如果同时指定了`name`和`type`，则从 Spring 上下文中找到唯一匹配的 bean 进行装配，找不到则抛出异常。
2. 如果指定了`name`，则从上下文中查找名称（id）匹配的 bean 进行装配，找不到则抛出异常。
3. 如果指定了`type`，则从上下文中找到类型匹配的唯一 bean 进行装配，找不到或是找到多个，都会抛出异常。
4. 如果既没有指定`name`，又没有指定`type`，则默认按照`byName`方式进行装配；如果没有匹配，按照`byType`进行装配。

Field injection is not recommended

在使用 IDEA 进行 Spring 开发的时候，当你在字段上面使用`@Autowired`注解的时候，你会发现 IDEA 会有警告提示：

> Field injection is not recommended Inspection info: Spring Team Recommends: "Always use constructor based dependency injection in your beans. Always use assertions for mandatory dependencies

![图片](https://mmbiz.qpic.cn/mmbiz/6fuT3emWI5KUJgwXhns2cguibibE2IJrbZgFpGgUXIiciahJysMvrDsHaQZ0eAfx2EiceBVV9KMGYP7qMEFMHmJ4kcQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

翻译过来就是这个意思：

> 不建议使用基于 field 的注入方式。Spring 开发团队建议：在你的Spring Bean 永远使用基于constructor 的方式进行依赖注入。对于必须的依赖，永远使用断言来确认。

比如如下代码：

```java
@Service
public class HelpService {
    @Autowired
    @Qualifier("svcB")
    private Svc svc;

    public void sayHello() {
        svc.sayHello();
    }
}
public interface Svc {
    void sayHello();
}
@Service
public class SvcB implements Svc {
    @Override
    public void sayHello() {
        System.out.println("hello, this is service B");
    }
}
```

将光标放到`@Autowired`处，使用`Alt + Enter` 快捷进行修改之后，代码就会变成基于 Constructor 的注入方式，修改之后：

```java
@Service
public class HelpService {
    private final Svc svc;
    @Autowired
    public HelpService(@Qualifier("svcB") Svc svc) {
        // Assert.notNull(svc, "svc must not be null");
        this.svc = svc;
    }
    public void sayHello() {
        svc.sayHello();
    }
}
```

如果按照 Spring 团队的建议，如果`svc`是必须的依赖，应该使用`Assert.notNull(svc, "svc must not be null")`来确认。

修正这个警告提示固然简单，但是我觉得更重要是去理解为什么 Spring 团队会提出这样的建议？直接使用这种基于 field 的注入方式有什么问题？

首先我们需要知道，Spring 中有这么3种依赖注入的方式：

- 基于 field 注入（属性注入）
- 基于 setter 注入
- 基于 constructor 注入（构造器注入）

### **1. 基于 field 注入**

所谓基于 field 注入，就是在bean的变量上使用注解进行依赖注入。本质上是通过反射的方式直接注入到 field。这是我平常开发中看的最多也是最熟悉的一种方式，同时，也正是 Spring 团队所不推荐的方式。比如：

```java
@Autowired
private Svc svc;
```

### **2. 基于 setter 方法注入**

通过对应变量的`setXXX()`方法以及在方法上面使用注解，来完成依赖注入。比如：

```java
private Helper helper;
@Autowired
public void setHelper(Helper helper) {
    this.helper = helper;
}
```

> 注：在 Spring 4.3 及以后的版本中，setter 上面的 @Autowired 注解是可以不写的。

### **3. 基于 constructor 注入**

将各个必需的依赖全部放在带有注解构造方法的参数中，并在构造方法中完成对应变量的初始化，这种方式，就是基于构造方法的注入。比如：

```java
private final Svc svc;
    
@Autowired
public HelpService(@Qualifier("svcB") Svc svc) {
    this.svc = svc;
}
```

> 在 Spring 4.3 及以后的版本中，如果这个类只有一个构造方法，那么这个构造方法上面也可以不写 @Autowired 注解。

基于 field 注入的好处

正如你所见，这种方式非常的简洁，代码看起来很简单，通俗易懂。你的类可以专注于业务而不被依赖注入所污染。你只需要把`@Autowired`扔到变量之上就好了，不需要特殊的构造器或者set方法，依赖注入容器会提供你所需的依赖。

#### 基于 field 注入的坏处

> 成也萧何败也萧何

基于 field 注入虽然简单，但是却会引发很多的问题。这些问题在我平常开发阅读项目代码的时候就经常遇见。

- 容易违背了单一职责原则 使用这种基于 field 注入的方式，添加依赖是很简单的，就算你的类中有十几个依赖你可能都觉得没有什么问题，普通的开发者很可能会无意识地给一个类添加很多的依赖。但是当使用构造器方式注入，到了某个特定的点，构造器中的参数变得太多以至于很明显地发现 something is wrong。拥有太多的依赖通常意味着你的类要承担更多的责任，明显违背了单一职责原则（SRP：Single responsibility principle）。

- 依赖注入与容器本身耦合

  依赖注入框架的核心思想之一就是受容器管理的类不应该去依赖容器所使用的依赖。换句话说，这个类应该是一个简单的 POJO(Plain Ordinary Java Object)能够被单独实例化并且你也能为它提供它所需的依赖。

  这个问题具体可以表现在：

- - 你的类不能绕过反射（例如单元测试的时候）进行实例化，必须通过依赖容器才能实例化，这更像是集成测试
  - 你的类和依赖容器强耦合，不能在容器外使用
  - 不能使用属性注入的方式构建不可变对象(`final` 修饰的变量)

#### Spring 开发团队的建议

> Since you can mix constructor-based and setter-based DI, it is a good rule of thumb to use constructors for mandatory dependencies and setter methods or configuration methods for optional dependencies.

简单来说，就是

- 强制依赖就用构造器方式

- 可选、可变的依赖就用 setter 注入

  当然你可以在同一个类中使用这两种方法。构造器注入更适合强制性的注入旨在不变性，Setter 注入更适合可变性的注入。

让我们看看 Spring 这样推荐的理由，首先是基于构造方法注入，

> ❝The Spring team generally advocates constructor injection as it enables one to implement application components as immutable objects and to ensure that required dependencies are not null. Furthermore constructor-injected components are always returned to client (calling) code in a fully initialized state. As a side note, a large number of constructor arguments is a bad code smell, implying that the class likely has too many responsibilities and should be refactored to better address proper separation of concerns.❞

Spring 团队提倡使用基于构造方法的注入，因为这样一方面可以将依赖注入到一个不可变的变量中 (注：`final` 修饰的变量)，另一方面也可以保证这些变量的值不会是 null。此外，经过构造方法完成依赖注入的组件 (注：比如各个 `service`)，在被调用时可以保证它们都完全准备好了。与此同时，从代码质量的角度来看，一个巨大的构造方法通常代表着出现了代码异味，这个类可能承担了过多的责任。

而对于基于 setter 的注入，他们是这么说的：

> ❝Setter injection should primarily only be used for optional dependencies that can be assigned reasonable default values within the class. Otherwise, not-null checks must be performed everywhere the code uses the dependency. One benefit of setter injection is that setter methods make objects of that class amenable to reconfiguration or re-injection later.❞

基于 setter 的注入，则只应该被用于注入非必需的依赖，同时在类中应该对这个依赖提供一个合理的默认值。如果使用 setter 注入必需的依赖，那么将会有过多的 null 检查充斥在代码中。使用 setter 注入的一个优点是，这个依赖可以很方便的被改变或者重新注入。 

## **小结**

以上就是本文的所有内容，希望阅读本文之后能让你对 Spring 的依赖注入有更深的理解。