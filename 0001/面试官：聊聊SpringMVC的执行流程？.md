# 面试官：聊聊SpringMVC的执行流程？

提起`Spring MVC`，你的第一印象是什么？

一个简化Web开发的轻量级框架？

实际上，现代开发过程中，开发流程与开发效率的不断提高，同时伴随着Restful与Json相结合的方式的兴起，使得多个设备跨平台的相互调用与访问变得简单了许多。

所以`Spring MVC`简化Web开发的使命也自然而然的变为了简化服务端开发。

那么今天我们就抛开繁杂的代码，从宏观的角度来看一看`Spring MVC`对于处理请求，简化服务端开发的解决方案是如何实现的。

## 1、曾经的王者——Servlet

在笔者刚接触到使用Java进行Web开发的时候，Spring MVC远没有今天这么流行，君不见曾经的王者Servlet繁盛一时的场面。现在回想起来，使用Servlet进行开发虽然不像现在这么容易，好多的事情需要自己做，但是Servlet使得开发的逻辑变得十分清晰，尤其是在Servlet与jsp很好的承担了各自的角色之后，再加上mvc分层思想的流行。编写Web应用程序在那时是一件快乐而又简单的事情。

实际上Servlet做的事情并不是很多，笔者觉得Servlet想要完成的就是统一请求的接受、处理与响应的流程。

> 网络编程中绕不开的一个东东想必不用说大家也猜得到，那就是`Socket`。但是网络需要传输的话是很复杂的，首先需要遵循一定的协议，现在我们一般使用Http与Https传输数据，而Socket就是在一些网络协议之上，屏蔽了底层协议的细节，为使用者提供一个统一的api。但是`Servlet`认为Socket做的还不够，或者说我们还要进行相应的处理。于是Servlet(就`HttpServlet`来说)，他将网络中的请求报文进行封装转化成为了Request表示，在`Http`通信过程之中就是`HttpServletRequest`，而将服务端处理请求后返回的响应统一的封装为了`HttpServletResponse`对象。

这样做的好处是什么呢？

我们作为开发者，不必再去做一些处理网络请求与响应的繁琐之事，而只需要关注于我们的业务逻辑开发。

> 大家有没有发现，每一次框架效率的提升很多时候都是在将最最重要的业务逻辑与其他任务尽可能完全的分离开，使我们总可以全身心的投入到业务逻辑的开发之中，Spring AOP是不是就是一个很好的佐证呢！

那么Servlet如何使用呢？

没有Servlet使用经历的同学可以听我简单的说一说：

1. 首先我们通常要编写一个自己的Servlet然后继承自HttpServlet,然后重写其doGet()与doPost()方法。这两个方法都会将HttpServletRequest与HttpServletResponse作为参数传递进去，然后我们从Request中提取前端传来的参数，在相应的doXXX方法内调用事先编写好的Service接口，Dao接口即可将数据准备好放置到Response中并跳转到指定的页面即可，跳转的方式可以选择转发或者重定向。
2. Servlet使用的是模板方法的设计模式，在Servlet顶层将会调用service方法，该方法会构造HttpServletRequest与HttpServletResponse对象作为参数调用子类重写的doXXX()方法。然后返回请求。
3. 最后我们需要将我们编写的自定义Servlet注册到web.xml中，在web.xml中配置servlet-mapping来为该servlet指定处理哪些请求。

Servlet的使用就是这么简单！事实上，在很长的一段时间内他的流行也得益于他的简单易用易上手。

```xml
<web-app xmlns="http://java.sun.com/xml/ns/j2ee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"
    version="2.4">
    <servlet>
            <servlet-name>ShoppingServlet</servlet-name>
            <servlet-class>com.myTest.ShoppingServlet</servlet-class>
    </servlet>

    <servlet-mapping>
            <servlet-name>ShoppingServlet</servlet-name>
            <url-pattern>/shop/ShoppingServlet</url-pattern>
    </servlet-mapping>
</web-app>
```

## 2、想要更进一步

当我们使用Servlet来进行业务逻辑开发的时候，时常会感觉到爽歪歪，但是爽歪歪的同时也感觉到有那么一点点不适。不适的地方主要有以下几点：

- 每个Servlet只能处理一个请求，这样当系统比较大，业务比较复杂的时候可能会存在成百上千的Servlet，找起来都眼花。
- 每次我们都需要手动的从Request中获取请求参数，然后封装成我们想要的对象，这其中可能还要对参数进行校验，在调用业务逻辑层获取到数据之后，我们还要手动的设置到响应中，同时手动的选择转发或者重定向进行跳转。
- 我们的请求的url是硬配置到web.xml中的，缺乏灵活性，如果可以动态的配置这种请求url与处理的对应关系就好了。
- 我们的Servlet与前端的渲染框架紧耦合在一块，这样当前端换一种显示技术的时候就需要改动较大的代码，如果能把数据的处理与数据的显示分离，让其松散耦合就更好了。

带着这些思考，能不能进一步的来抽离业务逻辑的开发呢？

在早期的时候笔者也曾进行一些尝试，其大概思路就是编写一个BaseServlet，然后我们自己定义的Servlet继承自BaseServlet，前端的请求需要指定Servlet的哪个方法进行处理，这样请求的时候将需要带上一个method参数，例如这样：

> http://localhost:8080/myProject/MyServlet?method=getInfo

在BaseServlet中将提取该参数信息，并使用反射的方法调用子类的该方法，子类方法统一返回String类型的结果，代表要返回的逻辑视图名，也就是要跳转的路径，然后父类拿到结果，使用重定向或者转发进行跳转。

> 说到这里，有小伙伴肯定不耐烦了，明明是讲Spring MVC的，到现在连个Spring MVC的影都还没见，全是在讲Servlet。先别着急，理解这些对我们理解Spring MVC有很大的帮助，请往下看

**\*说到这里，其实是想说，如果我们想要在Servlet上更进一步，想要进一步的将业务逻辑与其他工作相分离，那么就需要在Servlet之上，构建一个事无巨细，任劳任怨，神通过大，...(额想不起来。。。)的超级Servlet，来为我们做这些工作，我们暂且把这个Servlet叫做超级牛逼Servlet。而开发Spring的那些人是啥大佬，我们能想到这些，他们能想不到？于是他们动手开发了这个超级牛逼Servlet，并正式命名为DispatcherServlet。***

## 3、Spring MVC——两级控制器方式

接下来我们就要正式的开始Spring MVC之旅了，通过前面的了解，我们知道Spring MVC把那个超级牛逼Servlet叫做DispatcherServlet，这个Servlet可以说为简化我们的开发操碎了心，我们称之为_**前端控制器***。现在我们不禁思考，前面我们写的BaseServlet对应现在的超级牛逼Servlet(DispatcherServlet)。那么定义我们业务逻辑的自定义Servlet叫啥呢？Spring MVC管定义我们的业务逻辑处理的类叫做Handler，只不过他不再是一个Servlet了，而是一个普普通通的类，这也很好理解，毕竟DispatcherServlet做了太多，而且那么牛逼，完全可以像对待Servlet一样对待一个普通的类，而这个Handler就叫做***次级控制器**_。

> 这里可能有小伙伴持反对意见了，有的书上说了Spring MVC的次级控制器叫Controller，不是Handler。
>
> 其实Spring MVC的次级控制器确实是叫Handler，只不过Hander是一个抽象的，而Spring MVC选择使用Controller来实现Handler，讲到这里，你觉得我们能不能自定义一个Handler实现，叫做Lellortnoc呢？答案当然是可以的！就好像List是一个抽象的接口，而List的实现有ArrayList,LinkedList一样。

## 4、DispatcherServlet——前端控制器

DispatcherServlet是整个Spring MVC的核心，超级牛逼Servlet这个荣誉称号他是名副其实。DispatcherServlet和其家族成员兄弟一起完成了很多的工作，包括请求参数的自动绑定，参数的自动校验，请求url的自动匹配，逻辑视图名到真实页面的跳转，数据获取与数据渲染显示的分离等等。。。在此过程中他更像是一个指挥家，有条不紊的指挥着请求不断的向前处理，并最终完成服务端的响应数据。

想要了解具体DispatcherServlet都是怎么指挥的，那就继续往下看吧！推荐：[250期面试题汇总](http://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247489003&idx=1&sn=69bf19d900079e204e36df58525654bf&chksm=e80da39ddf7a2a8bf0765f9b95f359a3944fc40c4a192bb3fe9adedfbcd0070cd27234bcf6b3&scene=21#wechat_redirect)

## 5、HandlerMapper——请求映射专家

想想我们在使用Servlet编写代码的时候，请求的映射工作是交给了web.xml。但是现在Spring MVC采用了两级控制器的方式，就必须解决这个棘手的问题。

首先DispatcherServlet也是一个Servlet，那么我们也应该在web.xml中配置其处理的请求路径。那么应该配置什么路径呢？我们说DispatcherServlet被称为超级牛逼Serlvet，我们希望它能处理所有的请求，那么就可以让DispatcherServlet接受所有请求的处理。像下面这样配置：

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.0"
    xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
    http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
 <servlet>
          <servlet-name>Spring MVC</servlet-name>
          <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
          <!-- 表示启动容器时初始化该servlet -->
          <init-param>
              <param-name>contextConfigLocation</param-name>
              <param-value>classpath:Spring-servlet.xml</param-value>
          </init-param>
          <load-on-startup>1</load-on-startup>
     </servlet>
     <servlet-mapping>
          <servlet-name>Spring MVC</servlet-name>
          <url-pattern>/*</url-pattern>
     </servlet-mapping>

</web-app>
```

现在所有的请求都被映射到了DispatcherServlet，那么DispatcherServlet现在就有责任将请求分发至具体的次级控制器，如何找到或者说如何保存请求到具体的次级控制器的这种映射关系呢？DispatcherServlet选择请求他的好兄弟HandlerMapping。

在HandlerMapping中，保存了特定的请求url应该被哪一个Handler(也就是通常的Controller)所处理。HandlerMapping根据映射策略的不同，大概有下面几种映射查找方式：

1. org.springframework.web.servlet.handler.SimpleUrlHandlerMapping 通过配置请求路径和Controller映射建立关系，找到相应的Controller
2. org.springframework.web.servlet.mvc.support.ControllerClassNameHandlerMapping 通过 Controller 的类名找到请求的Controller。
3. org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping 通过定义的 beanName 进行查找要请求的Controller
4. org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping 通过注解 @RequestMapping(“/userlist”) 来查找对应的Controller。

想必现在最常用的就是第四种了吧，直接在对应的Controller上以及其内部的方法之上加上相应的注解，就可以配置好请求的映射，简直是香香的。推荐：[250期面试题汇总](http://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247489003&idx=1&sn=69bf19d900079e204e36df58525654bf&chksm=e80da39ddf7a2a8bf0765f9b95f359a3944fc40c4a192bb3fe9adedfbcd0070cd27234bcf6b3&scene=21#wechat_redirect)

## 6、Handler的拦路虎——HandlerInterceptor

聊到这里，你以为DispatcherServlet把请求的url交给HandlerMapping, HandlerMapping根据请求查出对应的Controller来交给DispatcherServlet, 然后DispatcherServlet交给Controller执行就完事了？那就To young to native了，这其中还有一些小插曲。比如我们不能啥请求不管三七二十一都交给Handler执行吧，最起码要过滤一下不合理的请求，比如跳转页面的时候检查Session,如果用户没登录跳转到登录界面啊，以及一些程序的异常以统一的方式跳转等等，都需要对请求进行拦截。

如果对Servlet了解的同学是不是有一点似曾相识的感觉？没错，Servlet中的Filter也可以完成请求拦截与过滤的功能，不过既然Spring MVC是两级控制器结构，那么HandlerInterceptor就与Filter有一些细微的差别，其最主要的差别，笔者认为HandlerInterceptor提供了更细粒度的拦截。毕竟Filter拦截的对象是Serlvet，而HandlerInterceptor拦截的则是Handler(Controller)。用一张图可以生动的表现出来。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XC7glibYd8yicJcJlib69gVL4UUsTibjIxLlICiagCqWDqlbMcBrgreyXdm7MCy5vSB1dbB6TibibySFic5aw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)HandlerInterceptor.jpg

从图中我们可以看出HandlerInteceptor可以配置多个，其中任何一个返回false的话，请求都将被拦截，直接返回。

## 7、次级控制器——Handler

前端控制器我们已经很熟悉了，而次级控制器也就是Handler，是我们真正执行业务逻辑的类。通常在Spring MVC中，这个Handler就是我们很熟悉的Controller。我们调用封装好的业务逻辑接口就是在这里进行处理的。可以说Spring MVC已经将业务逻辑与其他不相关的繁杂工作分离的较为彻底了。这样，我们就在Handler(Controller)中专心的编写我们的业务逻辑吧！

## 8、Handler与HandlerInterceptor的桥梁——HandlerExecutionChain

前面讲到DispatherServlet求助HandlerMapping进行url与次级控制器的映射，但是DispatherServlet在将url交给特定的HandlerMapping之后，HandlerMapping在进行了一顿猛如虎的操作之后，返回给DispaterServlet的却不是一个可执行的Handler(Controller),而是一个HandlerExecutionChain对象。那么HandlerMapping究竟为什么要返回给这样的一个对象而不是返回Handler对象呢？

其实在看上面图的时候，你有没有纳闷，HandlerInterceptor与Handler是怎样联系在一起的呢？答案就是HandlerExecutionChain。它就是若干的HandlerInterceptor与Handler的组合。那么是怎么组合的呢？

这里就涉及到设计模式中的责任链设计模式，HandlerExecutionChain将HandlerInterceptor与Handler串成一个执行链的形式，首先请求会被第一个HandlerInterceptor拦截，如果返回false,那么直接短路请求，如果返回true,那么再交给第二个HandlerInterceptor处理，直到所有的HandlerInterceptor都检查通过，请求才到达Handler(Controller)，交由Handler正式的处理请求。执行完成之后再逐层的返回。

而DispatcherServlet拿到的就是这样一个串联好的HandlerExecutionChain，然后顺序的执行请求。

## 9、解耦的关键——ModelAndView

到这里，请求终于来到了对应的Handler。我们希望的是Handler只处理负责的业务逻辑即可，而一些url的跳转等无需Handler负责。那么DispatcherServlet就使用了ModelAndView保存我们的数据和想要跳转的路径。

我们调用业务逻辑层获取数据，并将数据封装到ModelAndView中，同时设置ModelAndView的view逻辑视图名称。从ModelAndView的名称可以看出，它保存了Handler执行完成之后所需要发送到前端的数据，以及需要跳转的路径。这些是DispatcherServlet需要用到的。推荐：[250期面试题汇总](http://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247489003&idx=1&sn=69bf19d900079e204e36df58525654bf&chksm=e80da39ddf7a2a8bf0765f9b95f359a3944fc40c4a192bb3fe9adedfbcd0070cd27234bcf6b3&scene=21#wechat_redirect)

## 10、视图渲染查找——ViewResolver

这一步是Spring MVC将数据的获取与数据的显示渲染相分离的关键，前端可能采用各种各样的方式显示数据，可能是Jsp,可能是Html,也可能是其他的方式。DispatcherServlet已经拿到了ModelAndView，这里面有执行完成请求后返回的响应结果数据，还有逻辑视图的路径，这个时候DispatcherServlet就需要根据这个逻辑视图的路径去查找谁能把数据进行解析与渲染。

比如说我们使用FreeMarker模板引擎渲染数据，那么这个时候就要找到能够胜任该工作的那个View实现类，那么问题来了，如何寻找呢？以什么策略寻找呢？这个就依赖我们的ViewResolver了。

通常的寻找策略有以下几种：

- BeanNameViewResolver :将逻辑视图名解析为一个Bean,Bean的id等于逻辑视图名。
- XmlViewResolver:和BeanNameViewResolver类似，只不过目标视图Bean对象定义在一个独立的XML文件中，而非定义在DispatcherServlet上下文的主配置文件中
- InternalResourceViewResovlver:将视图名解析为一个URL文件，一般使用该解析器将视图名映射为保存在WEB-INF目录中的程序文件（如JSP）
- XsltViewResolver:将视图名解析为一个指定XSLT样式表的URL文件
- JasperReportsViewResolver:JasperReports是一个基于java的开源报表工具，该解析器将视图名解析为报表文件对应的URL
- FreeMarkerViewResolver:解析为基于FreeMarker模板技术的模板文件
- VelocityViewResolver和VelocityLayoutViewResolver:解析为基于Velocity模板技术的模板文件

## 11、数据渲染——View

在根据逻辑视图名借助ViewResolver查找到对应的View实现类之后，DispatcherServlet就会将ModelAndView中的数据交给View实现类来进行渲染，待该View渲染完成之后，会将渲染完成的数据交给DispatcherServlet，这时候DispatcherServlet将其封装到Response返回给前端显示。

至此，整个Spring MVC的处理流程就算完成了，当然这其中还会有对于国际化的支持，主题的定义与设置等等，但是这些是不常用的，Spring MVC最主要的处理流程所需要的用到的就是以上这些类。可以看到在此过程中，DispatcherServlet起到了至关重要的作用，所以说Spring MVC的核心就在于DispatcherServlet。

最后附上一张流程图作为以上内容的总结。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XC7glibYd8yicJcJlib69gVL4UzgOqlcicT1dbNcxem3OU7cIsSa4FdFezheI2ojLb5ejYKSuOAguoLWA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

 