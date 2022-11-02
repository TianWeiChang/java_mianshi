腾讯二面：@Bean与@Component用在同一个类上，会怎么样？

### **| 疑虑描述**

最近，在进行开发的过程中，发现之前的一个写法，类似如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxsh6uKqzgElbHsCVht9Fr0ibCHA0F6U5CkHJUggkCTy2kdBPwxaqKY5w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

以我的理解，@Configuration 加 @Bean 会创建一个 userName 不为 null 的 UserManager 对象，而 @Component 也会创建一个 userName 为 null 的 UserManager 对象。

**那么我们在其他对象中注入 UserManager 对象时，到底注入的是哪个对象？**

因为项目已经上线了很长一段时间了，所以这种写法没有编译报错，运行也没有出问题。后面去找同事了解下，实际是想让：

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxHLbSrNzibO0kCBG7pYytHSew4VTYdibLJibJWeUT8ibHtgr9ahKAagHPLA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

生效，而实际也确实是它生效了。那么问题来了：**Spring 容器中到底有几个 UserManager 类型的对象？**

### **|  Spring Boot 版本**

项目中用的 Spring Boot 版本是：2.0.3.RELEASE。对象的 scope 是默认值，也就是 singleton。

**结果验证**

验证方式有很多，可以 debug 跟源码，看看 Spring 容器中到底有几个 UserManager 对象，也可以直接从 UserManager 构造方法下手，看看哪几个构造方法被调用，等等。

我们从构造方法下手，看看 UserManager 到底实例化了几次。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyf7vEvw2CPyCOHMMkAVPkficV9nCjz8EmF8oB6POrRY77J2uWZxdDepw/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

只有有参构造方法被调用了，无参构造方法岿然不动（根本没被调用）。如果想了解的更深一点，可以读读：Spring 的循环依赖，源码详细分析 → 真的非要三级缓存吗？

> https://www.cnblogs.com/youzhibing/p/14337244.html

既然 UserManager 构造方法只被调用了一次，那么前面的问题：到底注入的是哪个对象。答案也就清晰了，没得选了呀，只能是 @Configuration 加 @Bean 创建的 userName 不为 null 的 UserManager 对象。

问题又来了：**为什么不是 @Component 创建的 userName 为 null 的 UserManager** 对象？

**源码解析**

@Configuration 与 @Component 关系很紧密。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxs0IYql8wZyDEjgQv0q0RRQJ369VzpUv87Vp984SUtKzsNbpMGcicntQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

所以@Configuration能够被component scan。

其中 ConfigurationClassPostProcessor与@Configuration 息息相关，其类继承结构图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxEzZWIl0MyFgfleMic1tzrbRRWgaSJm5TptKXjme5uHeXv6ibYedbGThA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

它实现了BeanFactoryPostProcessor接口和PriorityOrdered接口。

关于 BeanFactoryPostProcessor，可以看看：

> https://www.cnblogs.com/youzhibing/p/10559337.html

从AbstractApplicationContext的refresh方法调用的invokeBeanFactoryPostProcessors(beanFactory)开始，来跟下源码。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyxpGOSuia80cE2Ev2GkMfQVwEkUWKAGIJ3jBD588w0grcggOrcsXaM8w/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

此时完成了com.lee.qsl包下的component scan ，com.lee.qsl包及子包下的 UserConfig、UserController和UserManager都被扫描出来。注意，此刻@Bean的处理还未开始，UserManager是通过@Component而被扫描出来的；此时Spring容器中beanDefinitionMap中的 UserManager是这样的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxNoRvWVRhCIoRwjyTHHUWSp3ibehQJg9iaHCAKg8alusCgFG06Lj2oaHw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

接下来一步很重要，与我们想要的答案息息相关。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyLsM5mK6mE7LDRIZwMeKRe3B1IB1nZ76b92ya6icLefv0iaQiaAH2rTzvQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddx53Czf6GOy4MThcdUrn0cRVzwCI5fA2JQWQkNss10WQnS9hCPcTKs2A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

循环递归处理UserConfig 、UserController和UserManager，把它们都封装成 ConfigurationClass ，递归扫描 BeanDefinition。循环完之后，我们来看看 configClasses。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyruWUic1rCzQtiaibAGa3dp40IQrB14MmoQC0jbSMzvG4eCmxicoX2XgWXQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

UserConfig bean定义中beanMethods中有一个元素 [BeanMethod:name=userManager,declaringClass=com.lee.qsl.config.UserConfig]。

然后我们接着往下走，来仔细看看答案出现的环节。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyTqDfzlkmQMRgNlgKEwHONTXaHUY4ciaBgRRfQ92Bvtg0k6Ecz5iaNnbA/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

是不是有什么发现？@Component修饰的UserManager定义直接被覆盖成了@Configuration +@Bean修饰的UserManager定义。Bean定义类型也由ScannedGenericBeanDefinition替换成了ConfigurationClassBeanDefinition。

后续通过BeanDefinition创建实例的时候，创建的自然就是@Configuration+@Bean修饰的 UserManager，也就是会反射调用UserManager的有参构造方法。

自此，答案也就清楚了。Spring 其实给出了提示：

```
2021-10-03 20:37:33.697  INFO 13600 --- [           main] o.s.b.f.s.DefaultListableBeanFactory     : Overriding bean definition for bean 'userManager' with a different definition: replacing [Generic bean: class [com.lee.qsl.manager.UserManager]; scope=singleton; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in file [D:\qsl-project\spring-boot-bean-component\target\classes\com\lee\qsl\manager\UserManager.class]] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=userConfig; factoryMethodName=userManager; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [com/lee/qsl/config/UserConfig.class]]
```

只是日志级别是 info ，太不显眼了。

**Spring升级优化**

可能Spring团队意识到了info级别太不显眼的问题，或者说意识到了直接覆盖的处理方式不太合理。所以在Spring 5.1.2.RELEASE （Spring Boot 则是 2.1.0.RELEASE ）做出了优化处理。

我们来具体看看。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyf8cpGHEpWiab6bgg5NzWmBGC3XNA7pWyukH4N4NfXpGPQvGSFjvwL4Q/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

启动直接报错，Spring也给出了提示。

- 

```
The bean 'userManager', defined in class path resource [com/lee/qsl/config/UserConfig.class], could not be registered. A bean with that name has already been defined in file [D:\qsl-project\spring-boot-bean-component\target\classes\com\lee\qsl\manager\UserManager.class] and overriding is disabled.
```

我们来跟下源码，主要看看与Spring 5.0.7.RELEASE的区别。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyuZIDeEEV956jmOTny9OSY6285ZL8LKqAmeEF9DqXAdD7pAjtNP1vibw/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

新增了配置项allowBeanDefinitionOverriding来控制是否允许BeanDefinition覆盖，默认情况下是不允许的。我们可以在配置文件中配置：spring.main.allow-bean-definition-overriding=true ，允许BeanDefinition覆盖。这种处理方式是更优的，将选择权交给开发人员，而不是自己偷偷的处理，已达到开发者想要的效果。

**总 结**

Spring 5.0.7.RELEASE （ Spring Boot 2.0.3.RELEASE ）支持@Configuration+ @Bean与@Component同时作用于同一个类。启动时会给info级别的日志提示，同时会将@Configuration+@Bean修饰的 BeanDefinition覆盖掉@Component修饰的BeanDefinition。

也许Spring团队意识到了上述处理不太合适，于是在Spring 5.1.2.RELEASE做出了优化处理。增加了配置项：allowBeanDefinitionOverriding，将主动权交给了开发者，由开发者自己决定是否允许覆盖。

### **| 补充**

关于allowBeanDefinitionOverriding，前面有误，特意去翻了下源码，补充如下。Spring 1.2引进DefaultListableBeanFactory的时候就有了private boolean allowBeanDefinitionOverriding=true;，默认是允许BeanDefinition覆盖。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxcqBeYvIaKmMXRONuZzI7DfMkfUyq0EV0RL0E6e3mcKOZbusNeyicIMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Spring4.1.2引进isAllowBeanDefinitionOverriding()方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxWyge6TrTFRzJB6iaV0LCppzbeSbs8dqHj9PYoaicefVt4y3yOicv6PEVg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Spring自始至终默认都是允许BeanDefinition覆盖的，变的是Spring Boot ，Spring Boot 2.1.0之前没有覆盖Spring 的allowBeanDefinitionOverriding默认值，仍是允许BeanDefinition覆盖的。

Spring Boot 2.1.0中SpringApplication定义了私有属性：allowBeanDefinitionOverriding。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYNjqYHYfbGJPRFSp6YWddxSggNc7ck364KIic6bqIvlKzfQ6V24DVibcR2g6IJXkKrcfMbw1nfKLoQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)没有显示的指定值，那么默认值就是false ，之后在Spring Boot启动过程中，会用此值覆盖掉Spring中的allowBeanDefinitionOverriding的默认值。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbufOLaLdNcx76BmocY3GkWNyLUourSbWq84Mrk1SKwf21TnpyFmJY8z5EkjVNoFibO05icbCeFrhcrug/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

关于`allowBeanDefinitionOverriding`，我想大家应该已经清楚了。