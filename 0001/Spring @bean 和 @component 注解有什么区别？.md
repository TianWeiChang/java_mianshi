后端必备：10个Spring注解和区别

本文打算介绍几个不太容易说出其区别或者用途的 Spring 注解，比如 `@Component` 与 `@Bean` 的比较，`@ControllerAdvice` 是如何处理自定义异常的等等。

## 1. @Component 和 @Bean 的区别是什么？

1. 作用对象不同：`@Component` 注解作用于类，而 `@Bean` 注解作用于方法、
2. `@Component` 通常是通过路径扫描来自动侦测以及自动装配到 Spring 容器中(我们可以使用 `@ComponentScan` 注解定义要扫描的路径从中找出标识了需要装配的类自动装配到 Spring 的 bean 容器中)。`@Bean` 注解通常是我们在标有该注解的方法中定义产生这个 bean，`@Bean` 告诉了 Spring 这是某个类的实例，当我们需要用它的时候还给我。
3. `@Bean` 注解比 `@Component` 注解的自定义性更强，而且很多地方我们只能通过 `@Bean` 注解来注册 bean。比如当我们引用第三方库中的类需要装配到 Spring 容器时，只能通过 `@Bean` 来实现。

`@Bean` 注解使用示例：

```
@Configuration
public class AppConfig {
    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

@Component 注解使用示例：

```
@Component
public class ServiceImpl implements AService {
    ....
}
```

## 2. @Autowire 和 @Resource 的区别

1. `@Autowire` 和 `@Resource`都可以用来装配 bean，都可以用于字段或 setter 方法。
2. `@Autowire` 默认按类型装配，默认情况下必须要求依赖对象必须存在，如果要允许 null 值，可以设置它的 required 属性为 false。
3. `@Resource` 默认按名称装配，当找不到与名称匹配的 bean 时才按照类型进行装配。名称可以通过 name 属性指定，如果没有指定 name 属性，当注解写在字段上时，默认取字段名，当注解写在 setter 方法上时，默认取属性名进行装配。

> 注意：如果 name 属性一旦指定，就只会按照名称进行装配。

`@Autowire`和`@Qualifier`配合使用效果和`@Resource`一样：

```
@Autowired(required = false)
@Qualifier("example")
private Example example;

@Resource(name = "example")
private Example example;
```

`@Resource` 装配顺序

1. 如果同时指定 name 和 type，则从容器中查找唯一匹配的 bean 装配，找不到则抛出异常；
2. 如果指定 name 属性，则从容器中查找名称匹配的 bean 装配，找不到则抛出异常；
3. 如果指定 type 属性，则从容器中查找类型唯一匹配的 bean 装配，找不到或者找到多个抛出异常；
4. 如果不指定，则自动按照 byName 方式装配，如果没有匹配，则回退一个原始类型进行匹配，如果匹配则自动装配。

## 3. 将一个类声明为 Spring 的 bean 的注解有哪些？

- `@Component` ：通用的注解，可标注任意类为 Spring 的组件。如果一个 Bean 不知道属于哪个层，可以使用 `@Component` 注解标注。
- `@Repository` ：对应持久层即 Dao 层，主要用于数据库相关操作。
- `@Service` ：对应服务层，主要设计一些复杂的逻辑，需要用到 Dao 层。
- `@Controller` ：对应 Spring MVC 控制层，主要用来接受用户请求并调用 Service 层返回数据给前端页面。
- `@Configuration` ：声明该类为一个配置类，可以在此类中声明一个或多个 `@Bean` 方法。

## 4. @Configuration ：配置类注解

`@Configuration` 表明在一个类里可以声明一个或多个 `@Bean` 方法，并且可以由 Spring 容器处理，以便在运行时为这些 bean 生成 bean 定义和服务请求，例如：

```
@Configuration
public class AppConfig {
    @Bean
    public MyBean myBean() {
        // instantiate, configure and return bean ...
    }
}
```

我们可以通过 AnnotationConfigApplicationContext 来注册 `@Configuration` 类：

```
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.register(AppConfig.class);
ctx.refresh();
MyBean myBean = ctx.getBean(MyBean.class);
// use myBean ...
```

另外也可以通过组件扫描(component scanning)来加载，`@Configuration` 使用 `@Component` 进行原注解，因此 `@Configuration` 类也可以被组件扫描到(特别是使用 XML 的 元素)。`@Configuration` 类不仅可以使用组件扫描进行引导，还可以使用 `@ComponentScan` 注解自行配置组件扫描：

```
@Configuration
@ComponentScan("com.acme.app.services")
public class AppConfig {
    // various @Bean definitions ...
}
```

**使用 @Configuration 的约束：**

- 配置类必须以类的方式提供(比如不能是由工厂方法返回的实例)。
- 配置类必须是非 final 的。
- 配置类必须是非本地的(即可能不在方法中声明)，native 标注的方法。
- 任何嵌套的配置类必须声明为 static。
- @Bean 方法可能不会反过来创建更多的配置类。

除了单独使用 `@Configuration` 注解，我们还可以结合一些外部的 bean 或者注解共同使用，比如 `Environment API`，`@PropertySource`，`@Value`，`@Profile` 等等许多，这里就不做详细介绍了，更多的用法可以参看 Spring @Configuration 的相关文档 。

## 5. @ControllerAdvice ：处理全局异常利器

在 Spring 3.2 中，新增了 `@ControllerAdvice`、`@RestControllerAdvice`、`@RestController` 注解，可以用于定义 `@ExceptionHandler`、`@InitBinder`、`@ModelAttribute`，并应用到所有 `@RequestMapping` 、`@PostMapping`、`@GetMapping`等这些 Controller 层的注解中。

默认情况下，`@ControllerAdvice` 中的方法应用于全局所有的 Controller。而使用选择器 `annotations()`，`basePackageClasses()` 和 `basePackages()` (或其别名 value())来定义更小范围的目标 Controller 子集。

如果声明了多个选择器，则应用 OR 逻辑，这意味着所选的控制器应匹配至少一个选择器。请注意，选择器检查是在运行时执行的，因此添加许多选择器可能会对性能产生负面影响并增加复杂性。

`@ControllerAdvice` 我们最常使用的是结合 `@ExceptionHandler` 用于全局异常的处理。可以结合以下例子，我们可以捕获自定义的异常进行处理，并且可以自定义状态码返回：

```
@ControllerAdvice("com.developlee.errorhandle")
public class MyExceptionHandler {
    /**
     * 捕获CustomException
     * @param e
     * @return json格式类型
     */
    @ResponseBody
    @ExceptionHandler({CustomException.class}) //指定拦截异常的类型
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR) //自定义浏览器返回状态码
    public Map>String, Object< customExceptionHandler(CustomException e) {
        Map<String, Object> map = new HashMap<>();
        map.put("code", e.getCode());
        map.put("msg", e.getMsg());
        return map;
    }
}
```

更多信息可以参看 Spring @ControllerAdvice 的官方文档。

## 6. @Component, @Repository, @Service 的区别

`@Component`是一个通用的 Spring 容器管理的单例 bean 组件。而`@Repository`, `@Service`, `@Controller`就是针对不同的使用场景所采取的特定功能化的注解组件。

因此，当你的一个类被@Component 所注解，那么就意味着同样可以用`@Repository`, `@Service`, `@Controller` 来替代它，同时这些注解会具备有更多的功能，而且功能各异。

最后，如果你不知道要在项目的业务层采用`@Service`还是`@Component`注解。那么，`@Service`是一个更好的选择。

## 7.总结

以上简单介绍了几种 Spring 中的几个注解及代码示例，就我个人而言，均是平时用到且不容易理解的几个，或者容易忽略的几个。当然，这篇文章并没有完全介绍完，在今后还会继续补充完善。