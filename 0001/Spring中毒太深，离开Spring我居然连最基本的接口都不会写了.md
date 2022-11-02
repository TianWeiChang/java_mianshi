Spring中毒太深，离开Spring我居然连最基本的接口都不会写了

随着 `Spring` 的崛起以及其功能的完善，现在可能绝大部分项目的开发都是使用 `Spring（全家桶）` 来进行开发，`Spring`也确实和其名字一样，是开发者的春天，`Spring` 解放了程序员的双手，而等到 `SpringBoot`出来之后配置文件大大减少，更是进一步解放了程序员的双手，但是也正是因为`Spring`家族产品的强大，使得我们习惯了面向 `Spring` 开发，那么假如有一天没有了 `Spring`，是不是感觉心里一空，可能一下子连最基本的接口都不会写了，尤其是没有接触过`Servlet`编程的朋友。因为加入没有了 `Spring` 等框架，那么我们就需要利用最原生的 `Servlet` 来自己实现接口路径的映射，对象也需要自己进行管理。

## Spring 能帮我们做什么

`Spring` 是为解决企业级应用开发的复杂性而设计的一款框架，`Spring` 的设计理念就是：简化开发。

在 `Spring` 框架中，一切对象都是 `bean`，所以其通过面向 `bean` 编程（BOP），结合其核心思想依赖注入（DI）和面向切面(（AOP）编程，`Spring` 实现了其伟大的简化开发的设计理念。

### 控制反转（IOC）

`IOC` 全称为：Inversion of Control。控制反转的基本概念是：不用创建对象，但是需要描述创建对象的方式。

简单的说我们本来在代码中创建一个对象是通过 `new` 关键字，而使用了 `Spring` 之后，我们不在需要自己去 `new` 一个对象了，而是直接通过容器里面去取出来，再将其自动注入到我们需要的对象之中，即：依赖注入。

也就说创建对象的控制权不在我们程序员手上了，全部交由 `Spring` 进行管理，程序要只需要注入就可以了，所以才称之为控制反转。

### 依赖注入（DI）

依赖注入（Dependency Injection，DI）就是 `Spring` 为了实现控制反转的一种实现方式，所有有时候我们也将控制反转直接称之为依赖注入。

### 面向切面编程（AOP）

`AOP` 全称为：Aspect Oriented Programming。`AOP`是一种编程思想，其核心构造是方面（切面），即将那些影响多个类的公共行为封装到可重用的模块中，而使原本的模块内只需关注自身的个性化行为。

`AOP` 编程的常用场景有：Authentication（权限认证）、Auto Caching（自动缓存处理）、Error Handling（统一错误处理）、Debugging（调试信息输出）、Logging（日志记录）、Transactions（事务处理）等。

### 利用 Spring 来完成 Hello World

最原生的 `Spring` 需要较多的配置文件，而 `SpringBoot` 省略了许多配置，相比较于原始的 `Spring` 又简化了不少，在这里我们就以 `SpringBoot` 为例来完成一个简单的接口开发。

- 1、新建一个 `maven` 项目，`pom` 文件中引入依赖（省略了少部分属性）：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.4.0</version>
    <relativePath/>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

- 2、新建一个 `HelloController` 类：

```java
package com.lonely.wolf.note.springboot.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/hello")
public class HelloController {
    @GetMapping("/demo")
    public String helloWorld(String name){
        return "Hello：" + name;
    }
}
```

- 3、最后新建一个 `SpringBoot` 启动类：

```java
package com.lonely.wolf.note.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication(scanBasePackages = "com.lonely.wolf.note.springboot")
class MySpringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(MySpringBootApplication.class, args);
    }
}
```

- 4、现在就可以输入测试路径：`http://localhost:8080/hello/demo?name=双子孤狼` 进行测试，正常输出：`Hello：双子孤狼`。

我们可以看到，利用 `SpringBoot` 来完成一个简单的应用开发非常简单，可以不需要任何配置完成一个简单的应用，这是因为 `SpringBoot` 内部已经做好了约定（约定优于配置思想），包括容器 `Tomcat` 都被默认集成，所以我们不需要任何配置文件就可以完成一个简单的 `demo` 应用。

## 假如没有了 Spring

通过上面的例子我们可以发现，利用 `Spring` 来完成一个 `Hello World` 非常简单，但是假如没有了 `Spring`，我们又该如何完成这样的一个 `Hello World` 接口呢？

### 基于 Servlet 开发

在还没有框架之前，编程式基于原始的 `Servlet` 进行开发，下面我们就基于原生的 `Servlet` 来完成一个简单的接口调用。

- 1、`pom` 文件引入依赖，需要注意的是，`package` 属性要设置成 `war` 包，为了节省篇幅，这里没有列出 `pom` 完整的信息：

```xml
<packaging>war</packaging> 
<dependencies>
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>servlet-api</artifactId>
        <version>2.4</version>
    </dependency>

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.7</version>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.72</version>
    </dependency>
</dependencies>
```

- 2、在 `src/main` 下面新建文件夹 `webapp/WEB-INF`，然后在 `WEB-INF` 下面新建一个 `web.xml` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://java.sun.com/xml/ns/j2ee" xmlns:javaee="http://java.sun.com/xml/ns/javaee"
	xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"
	version="2.4">
	<display-name>Lonely Wolf Web Application</display-name>
	<servlet>
		<servlet-name>helloServlet</servlet-name>
		<servlet-class>com.lonely.wolf.mini.spring.servlet.HelloServlet</servlet-class>
	</servlet>
	<servlet-mapping>
		<servlet-name>helloServlet</servlet-name>
		<url-pattern>/hello/*</url-pattern>
	</servlet-mapping>
</web-app>
```

这里面定义了 `selvlet` 和 `servlet-mapping` 两个标签，这两个标签必须一一对应，上面的标签定义了 `servlet` 的位置，而下面的 `servlet-mapping` 文件定义了路径的映射，这两个标签通过 `servlet-name` 标签对应。

- 3、新建一个 `HelloServlet` 类继承 `HttpServlet`：

```java
package com.lonely.wolf.mini.spring.servlet;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * 原始Servlet接口编写，一般需要实现GET和POST方法，其他方法可以视具体情况选择性继承
 */
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doPost(request,response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/html;charset=utf-8");
        response.getWriter().write("Hello：" + request.getParameter("name"));
    }
}
```

- 4、执行 `maven` 打包命令，确认成功打包成 `war` 包：

![img](https://img2020.cnblogs.com/blog/2232223/202012/2232223-20201213102058355-972685274.png)

- 5、`RUN-->Edit Configurations`，然后点击左上角的 `+` 号，新建一个 `Tomcat Server`，如果是第一次配置，默认没有 `Tomcat Server` 选项，需要点击底部的 `xx more items...`：

![img](https://img2020.cnblogs.com/blog/2232223/202012/2232223-20201213102124749-1680335127.png)

- 6、点击右边的 `Deployment`，然后按照下图依次点击，最后在弹框内找到上面打包好的 `war` 包文件：

![img](https://img2020.cnblogs.com/blog/2232223/202012/2232223-20201213102147338-1168179967.png)

- 7、选中之后，需要注意的是，下面 `Application Context` 默认会带上 `war` 包名，为了方便，我们需要把它删掉，即不用上下文路径，只保留一个根路径 `/` （当然上下文也可以保留，但是每次请求都要带上这一部分）， 再选择 `Apply`，点击 `OK`，即可完成部署：

![img](https://img2020.cnblogs.com/blog/2232223/202012/2232223-20201213102207412-1033660516.png)

- 8、最后我们在浏览器输入请求路径`http://localhost:8080/hello?name=双子孤狼`，即可得到返回：`Hello：双子孤狼`。

上面我们就完成了一个简单的 基于`Servlet` 的接口开发，可以看到，配置非常麻烦，每增加一个 `Servlet` 都需要增加对应的配置，所以才会有许多框架的出现来帮我们简化开发，比如原来很流行的 `Struts2` 框架，当然现在除了一些比较老的项目，一般我们都很少使用，而更多的是选择 `Spring` 框架来进行开发。

### 模仿Spring

`Spring` 的源码体系非常庞大，大部分人对其源码都敬而远之。确实，`Spring` 毕竟经过了这么多年的迭代，功能丰富，项目庞大，不是一下子就能看懂的。虽然 `Spring` 难以理解，但是其最核心的思想仍然是我们上面介绍的几点，接下来就基于 `Spring` 最核心的部分来模拟，自己动手实现一个超级迷你版本的 `Spring`（此版本并不包含 `AOP` 功能）。

- 1、`pom` 依赖和上面保持不变，然后 `web.xml` 作如下改变，这里会拦截所有的接口 `/*`，然后多配置了一个参数，这个参数其实也是为了更形象的模拟 `Spring`：

```xml
<servlet>
    <servlet-name>myDispatcherServlet</servlet-name>
    <servlet-class>com.lonely.wolf.mini.spring.v1.MyDispatcherServlet</servlet-class>
    <init-param>
        <param-name>defaultConfig</param-name>
        <param-value>application.properties</param-value>
    </init-param>
</servlet>

<servlet-mapping>
    <servlet-name>myDispatcherServlet</servlet-name>
    <url-pattern>/*</url-pattern>
</servlet-mapping>
```

- 2、在 `respurces` 下面新建一个配置文件 `application.properties`，用来定义扫描的基本路径：

```properties
basePackages=com.lonely.wolf.mini.spring
```

- 3、创建一些相关的注解类：

```java
package com.lonely.wolf.mini.spring.annotation;

import java.lang.annotation.*;

@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WolfAutowired {
    String value() default "";
}
```

```java
package com.lonely.wolf.mini.spring.annotation;

import java.lang.annotation.*;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WolfController {
    String value() default "";
}
```

```java
package com.lonely.wolf.mini.spring.annotation;

import java.lang.annotation.*;

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WolfGetMapping {
    String value() default "";
}
```

```java
package com.lonely.wolf.mini.spring.annotation;

import java.lang.annotation.*;

@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WolfRequestParam {
    String value() default "";
}
```

```java
package com.lonely.wolf.mini.spring.annotation;

import java.lang.annotation.*;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WolfService {
    String value() default "";
}
```

- 4、这个时候最核心的逻辑就是 `MyDispatcherServlet` 类了：

```java
package com.lonely.wolf.mini.spring.v1;

import com.lonely.wolf.mini.spring.annotation.*;
import com.lonely.wolf.mini.spring.v1.config.MyConfig;
import org.apache.commons.lang3.StringUtils;

import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.net.URL;
import java.util.*;

public class MyDispatcherServlet extends HttpServlet {
    private MyConfig myConfig = new MyConfig();
    private List<String> classNameList = new ArrayList<String>();

    private Map<String,Object> iocContainerMap = new HashMap<>();
    private Map<String,HandlerMapping> handlerMappingMap = new HashMap<>();

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doPost(request,response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        try {
            this.doDispatch(request, response);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception{
        String requestUrl = this.formatUrl(request.getRequestURI());
        HandlerMapping handlerMapping = handlerMappingMap.get(requestUrl);
        if (null == handlerMapping){
            response.getWriter().write("404 Not Found");
            return;
        }

        //获取方法中的参数类型
        Class<?>[] paramTypeArr = handlerMapping.getMethod().getParameterTypes();
        Object[] paramArr = new Object[paramTypeArr.length];

        for (int i=0;i<paramTypeArr.length;i++){
            Class<?> clazz = paramTypeArr[i];
            //参数只考虑三种类型，其他不考虑
            if (clazz == HttpServletRequest.class){
                paramArr[i] = request;
            }else if (clazz == HttpServletResponse.class){
                paramArr[i] = response;
            } else if (clazz == String.class){
                Map<Integer,String> methodParam = handlerMapping.getMethodParams();
                paramArr[i] = request.getParameter(methodParam.get(i));
            }else{
                System.out.println("暂不支持的参数类型");
            }
        }
        //反射调用controller方法
        handlerMapping.getMethod().invoke(handlerMapping.getTarget(), paramArr);
    }

    private String formatUrl(String requestUrl) {
        requestUrl = requestUrl.replaceAll("/+","/");
        if (requestUrl.lastIndexOf("/") == requestUrl.length() -1){
            requestUrl = requestUrl.substring(0,requestUrl.length() -1);
        }
        return requestUrl;
    }


    @Override
    public void init(ServletConfig config) throws ServletException {
        //1.加载配置文件
        try {
            doLoadConfig(config.getInitParameter("defaultConfig"));
        } catch (Exception e) {
            System.out.println("加载配置文件失败");
            return;
        }

        //2.根据获取到的扫描路径进行扫描
        doScanPacakge(myConfig.getBasePackages());

        //3.将扫描到的类进行初始化，并存放到IOC容器
        doInitializedClass();

        //4.依赖注入
        doDependencyInjection();

        System.out.println("DispatchServlet Init End..." );
    }


    private void doDependencyInjection() {
        if (iocContainerMap.size() == 0){
            return;
        }
        //循环IOC容器中的类
        Iterator<Map.Entry<String,Object>> iterator = iocContainerMap.entrySet().iterator();

        while (iterator.hasNext()){
            Map.Entry<String,Object> entry = iterator.next();
            Class<?> clazz = entry.getValue().getClass();
            Field[] fields = clazz.getDeclaredFields();

            //属性注入
            for (Field field : fields){
                //如果属性有WolfAutowired注解则注入值（暂时不考虑其他注解）
                if (field.isAnnotationPresent(WolfAutowired.class)){
                    String value = toLowerFirstLetterCase(field.getType().getSimpleName());//默认bean的value为类名首字母小写
                    if (field.getType().isAnnotationPresent(WolfService.class)){
                        WolfService wolfService = field.getType().getAnnotation(WolfService.class);
                        value = wolfService.value();
                    }
                    field.setAccessible(true);
                    try {
                        Object target = iocContainerMap.get(beanName);
                        if (null == target){
                            System.out.println(clazz.getName() + "required bean:" + beanName + ",but we not found it");
                        }
                        field.set(entry.getValue(),iocContainerMap.get(beanName));//初始化对象，后面注入
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }

            //初始化HanderMapping
            String requestUrl = "";
            //获取Controller类上的请求路径
            if (clazz.isAnnotationPresent(WolfController.class)){
                requestUrl = clazz.getAnnotation(WolfController.class).value();
            }

            //循环类中的方法，获取方法上的路径
            Method[] methods = clazz.getMethods();
            for (Method method : methods){
                //假设只有WolfGetMapping这一种注解
                if(!method.isAnnotationPresent(WolfGetMapping.class)){
                    continue;
                }
                WolfGetMapping wolfGetMapping = method.getDeclaredAnnotation(WolfGetMapping.class);
                requestUrl = requestUrl + "/" + wolfGetMapping.value();//拼成完成的请求路径

                //不考虑正则匹配路径/xx/* 的情况，只考虑完全匹配的情况
                if (handlerMappingMap.containsKey(requestUrl)){
                    System.out.println("重复路径");
                    continue;
                }

                Annotation[][] annotationArr = method.getParameterAnnotations();//获取方法中参数的注解

                Map<Integer,String> methodParam = new HashMap<>();//存储参数的顺序和参数名
                retryParam:
                for (int i=0;i<annotationArr.length;i++){
                    for (Annotation annotation : annotationArr[i]){
                        if (annotation instanceof WolfRequestParam){
                            WolfRequestParam wolfRequestParam = (WolfRequestParam) annotation;
                            methodParam.put(i,wolfRequestParam.value());//存储参数的位置和注解中定义的参数名
                            continue retryParam;
                        }
                    }
                }

                requestUrl = this.formatUrl(requestUrl);//主要是防止路径多了/导致路径匹配不上
                HandlerMapping handlerMapping = new HandlerMapping();
                handlerMapping.setRequestUrl(requestUrl);//请求路径
                handlerMapping.setMethod(method);//请求方法
                handlerMapping.setTarget(entry.getValue());//请求方法所在controller对象
                handlerMapping.setMethodParams(methodParam);//请求方法的参数信息
                handlerMappingMap.put(requestUrl,handlerMapping);//存入hashmap
            }
        }
    }


    /**
     * 初始化类，并放入容器iocContainerMap内
     */
    private void doInitializedClass() {
        if (classNameList.isEmpty()){
            return;
        }
        for (String className : classNameList){
            if (StringUtils.isEmpty(className)){
                continue;
            }
            Class clazz;
            try {
                clazz = Class.forName(className);//反射获取对象
                if (clazz.isAnnotationPresent(WolfController.class)){
                    String value = ((WolfController)clazz.getAnnotation(WolfController.class)).value();
                    //如果直接指定了value则取value，否则取首字母小写类名作为key值存储类的实例对象
                    iocContainerMap.put(StringUtils.isBlank(value) ? toLowerFirstLetterCase(clazz.getSimpleName()) : value,clazz.newInstance());
                }else if(clazz.isAnnotationPresent(WolfService.class)){
                    String value = ((WolfService)clazz.getAnnotation(WolfService.class)).value();
                    iocContainerMap.put(StringUtils.isBlank(value) ? toLowerFirstLetterCase(clazz.getSimpleName()) : value,clazz.newInstance());
                }else{
                    System.out.println("不考虑其他注解的情况");
                }
            } catch (Exception e) {
                e.printStackTrace();
                System.out.println("初始化类失败，className为" + className);
            }
        }

    }

    /**
     * 将首字母转换为小写
     * @param className
     * @return
     */
    private String toLowerFirstLetterCase(String className) {
        if (StringUtils.isBlank(className)){
            return "";
        }
        String firstLetter = className.substring(0,1);
        return firstLetter.toLowerCase() + className.substring(1);
    }


    /**
     * 扫描包下所有文件获取全限定类名
     * @param basePackages
     */
    private void doScanPacakge(String basePackages) {
        if (StringUtils.isBlank(basePackages)){
            return;
        }
        //把包名的.替换为/
        String scanPath = "/" + basePackages.replaceAll("\\.","/");
        URL url = this.getClass().getClassLoader().getResource(scanPath);//获取到当前包所在磁盘的全路径
        File files = new File(url.getFile());//获取当前路径下所有文件
        for (File file : files.listFiles()){//开始扫描路径下的所有文件
            if (file.isDirectory()){//如果是文件夹则递归
                doScanPacakge(basePackages + "." + file.getName());
            }else{//如果是文件则添加到集合。因为上面是通过类加载器获取到的文件路径，所以实际上是class文件所在路径
                classNameList.add(basePackages + "." + file.getName().replace(".class",""));
            }
        }

    }


    /**
     * 加载配置文件
     * @param configPath - 配置文件所在路径
     */
    private void doLoadConfig(String configPath) {
        InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream(configPath);
        Properties properties = new Properties();
        try {
            properties.load(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
            System.out.println("加载配置文件失败");
        }

        properties.forEach((k, v) -> {
            try {
                Field field = myConfig.getClass().getDeclaredField((String)k);
                field.setAccessible(true);
                field.set(myConfig,v);
            } catch (Exception e) {
                e.printStackTrace();
                System.out.println("初始化配置类失败");
                return;
            }
        });
    }
}
```

- 5、这个 `Servlet` 相比较于上面的 `HelloServlet` 多了一个 `init` 方法，这个方法中主要做了以下几件事情：

（1）初始化配置文件，拿到配置文件中配置的参数信息（对应方法：`doLoadConfig`）。

（2）拿到第 `1` 步加载出来的配置文件，获取到需要扫描的包路径，然后将包路径进行转换成实际的磁盘路径，并开始遍历磁盘路径下的所有 `class` 文件，最终经过转换之后得到扫描路径下的所有类的全限定类型，存储到全局变量 `classNameList` 中（对应方法：`doScanPacakge`）。

（3）根据第 `2` 步中得到的全局变量 `classNameList` 中的类通过反射进行初始化（需要注意的是只会初始化加了指定注解的类）并将得到的对应关系存储到全局变量 `iocContainerMap` 中（即传说中的 `IOC` 容器），其中 `key` 值为注解中的 `value` 属性，如 `value` 属性为空，则默认取首字母小写的类名作为 `key` 值进行存储（对应方法：`doInitializedClass`）。

（4）这一步比较关键，需要对 `IOC` 容器中的所有类的属性进行赋值并且需要对 `Controller` 中的请求路径进行映射存储，为了确保最后能顺利调用 `Controller` 中的方法，还需要将方法的参数进行存储 。对属性进行映射时只会对加了注解的属性进行映射，映射时会从 `IOC` 容器中取出第 `3` 步中已经初始化的实例对象进行赋值，最后将请求路径和 `Controller` 中方法的映射关系存入变量 `handlerMappingMap`，`key` 值为请求路径，`value` 为方法的相关信息 （对应方法：`doDependencyInjection`）。

- 6、存储请求路径和方法的映射关系时，需要用到 `HandlerMapping` 类来进行存储：

```java
package com.lonely.wolf.mini.spring.v1;

import java.lang.reflect.Method;
import java.util.Map;

//省略了getter/setter方法
public class HandlerMapping {
    private String requestUrl;
    private Object target;//保存方法对应的实例
    private Method method;//保存映射的方法
    private Map<Integer,String> methodParams;//记录方法参数
}
```

- 7、初始化完成之后，因为拦截了 `/*` ，所以调用任意接口都会进入 `MyDispatcherServlet` ，而且最终都会执行方法 `doDispatch`，执行这个方法时会拿到请求的路径，然后和全局变量 `handlerMappingMap` 进行匹配，匹配不上则返回 `404`，匹配的上则取出必要的参数进行赋值，最后通过反射调用到 `Controller` 中的相关方法。
- 8、新建一个 `HelloController` 和 `HelloService` 来进行测试：

```java
package com.lonely.wolf.mini.spring.controller;

import com.lonely.wolf.mini.spring.annotation.WolfAutowired;
import com.lonely.wolf.mini.spring.annotation.WolfController;
import com.lonely.wolf.mini.spring.annotation.WolfGetMapping;
import com.lonely.wolf.mini.spring.annotation.WolfRequestParam;
import com.lonely.wolf.mini.spring.service.HelloService;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WolfController
public class HelloController {
    @WolfAutowired
    private HelloService helloService;

    @WolfGetMapping("/hello")
    public void query(HttpServletRequest request,HttpServletResponse response, @WolfRequestParam("name") String name) throws IOException {
        response.setContentType("text/html;charset=utf-8");
        response.getWriter().write("Hello：" + name);
    }
}
```

```java
package com.lonely.wolf.mini.spring.service;

import com.lonely.wolf.mini.spring.annotation.WolfService;

@WolfService(value = "hello_service")//为了演示能否正常取value属性
public class HelloService {
}
```

- 9、输入测试路径：`http://localhost:8080////hello?name=双子孤狼`， 进行测试发现可以正常输出：`Hello：双子孤狼`。

上面这个例子只是一个简单的演示，通过这个例子只是希望在没有任何框架的情况下，我们也能知道如何完成一个简单的应用开发。例子中很多细节都没有进行处理，仅仅只是为了体验一下 `Spring` 的核心思想，并了解 `Spring` 到底帮助我们做了什么，实际上 `Spring` 能帮我们做的事情远比这个例子中多得多，`Spring` 体系庞大，设计优雅，经过了多年的迭代优化，是一款非常值得研究的框架。

## 总结

本文从介绍 `Spring` 核心功能开始入手，从如何利用 `Spring` 完成一个应用开发，讲述到假如没有 `Spring` 我们该如何基于 `Servlet` 进行开发，最后再通过一个简单的例子体验了 `Spring` 的核心思想。