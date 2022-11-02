手写SpringMVC框架



## 写在前面

Spring 想必大家都听说过，可能现在更多流行的是Spring Boot 和Spring Cloud 框架；但是`SpringMVC` 作为一款实现了`MVC` 设计模式的web (表现层) 层框架，其高开发效率和高性能也是现在很多公司仍在采用的框架；除此之外，Spring 源码大师级的代码规范和设计思想都十分值得学习；退一步说，`SpringMVC`框架底层也有很多Spring 的东西，而且面试的时候还会经常被问到`SpringMVC`原理，一般人可能也就是只能把`SpringMVC` 的运行原理背出来罢了，至于问到有没有了解其底层实现(代码层面)，那很可能就歇菜了，但您要是可以手写`SpringMVC `框架就肯定可以令面试官刮目相看，所以手写`SpringMVC` 值得一试。

在设计自己的`SpringMVC`框架之前，需要了解下其运行流程。

## 一、SpringMVC 运行流程

![图1. SpringMVC 运行流程](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109221814638-1557302595.jpg) 

通过上面流程图，我们总结为以下几个步骤：

1、用户向服务器发送请求，请求被Spring 前端控制器`DispatcherServlet `捕获;

2、`DispatcherServlet `收到请求后调用`HandlerMapping `处理器映射器；

3、处理器映射器对请求URL 进行解析，得到请求资源标识符(URI)；然后根据该URI，调用`HandlerMapping `获得该Handler 配置的所有相关的对象(包括Handler 对象以及Handler 对象对应的拦截器)，再以`HandlerExecutionChain` 对象的形式返回给`DispatcherServlet`；

4、`DispatcherServlet `根据获得的Handler，通过`HandlerAdapter` 处理器适配器选择一个合适的`HandlerAdapter`；(附注:如果成功获得`HandlerAdapter` 后，此时将开始执行拦截器的`preHandler(...)`方法)；

5、提取Request 中的模型数据，填充Handler 入参，开始执行Handler(即Controller)；【在填充Handler的入参过程中，根据你的配置，Spring 将帮你做一些额外的工作如：`HttpMessageConveter`：将请求消息(如Json、xml等数据)转换成一个对象，将对象转换为指定的响应信息；数据转换：对请求消息进行数据转换，如String转换成Integer、Double等；数据格式化：对请求消息进行数据格式化，如将字符串转换成格式化数字或格式化日期等；数据验证：验证数据的有效性(长度、格式等)，验证结果存储到`BindingResult`或Error中 】

6、Controller 执行完成返回`ModelAndView`对象；

7、HandlerAdapter 将controller 执行结果ModelAndView 对象返回给DispatcherServlet；

8、`DispatcherServlet `将ModelAndView 对象传给ViewReslover 视图解析器；

9、`ViewReslover` 根据返回的ModelAndView，选择一个适合的ViewResolver (必须是已经注册到Spring容器中的ViewResolver)返回给DispatcherServlet；

10、`DispatcherServlet `对View 进行渲染视图（即将模型数据填充至视图中)；

11、`DispatcherServlet `将渲染结果响应用户(客户端)。



## 二、SpringMVC 框架设计思路

#### 1、读取配置阶段

 ![图2. SpringMVC 继承关系](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109215313253-1976289286.png) 

 第一步就是配置`web.xml`，加载自定义的`DispatcherServlet`。而从图中可以看出，`Spring MVC` 本质上是一个`Servlet`，这个`Servlet `继承自`HttpServlet`，此外，`FrameworkServlet `负责初始`Spring MVC`的容器，并将Spring 容器设置为父容器；为了读取`web.xml `中的配置，需要用到`ServletConfig` 这个类，它代表当前Servlet 在web.xml 中的配置信息，然后通过`web.xml `中加载我们自己写的`MyDispatcherServlet `和读取配置文件。 

#### 2、初始化阶段

初始化阶段会在`DispatcherServlet` 类中，按顺序实现下面几个步骤：

1、加载配置文件；

2、扫描当前项目下的所有文件；

3、拿到扫描到的类，通过反射机制将其实例化，并且放到ioc 容器中(Map的键值对  beanName-bean) beanName默认是首字母小写；

4、初始化path 与方法的映射；

5、获取请求传入的参数并处理参数通过初始化好的`handlerMapping` 中拿出`url `对应的方法名，反射调用。

#### 3、运行阶段

运行阶段，每一次请求将会调用`doGet` 或`doPost` 方法，它会根据`url` 请求去`HandlerMapping `中匹配到对应的Method，然后利用反射机制调用Controller 中的url 对应的方法，并得到结果返回。 

## 实现Spring  MVC 框架

首先，`Spring MVC` 框架只实现自己的`@Controller `和`@RequestMapping `注解，其它注解功能实现方式类似，实现注解较少所以项目比较简单，可以看到如下工程文件及目录截图。 

![项目目录](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109222945934-40583490.png) 

#### 创建Java Web项目

创建Java Web 项目，勾选JavaEE 下方的Web Application 选项，Next。 

![创建Java Web 项目](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109223232883-59709633.png) 

#### 配置maven

配置一个pom文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <servlet>
        <servlet-name>DispatcherServlet</servlet-name>
        <servlet-class>com.tjt.springmvc.DispatcherServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>DispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

#### 创建自定义的controller注解

```java
package com.tjt.springmvc;


import java.lang.annotation.*;


/**
 * @MyController 自定义注解类
 *
 * @@Target(ElementType.TYPE)
 * 表示该注解可以作用在类上;
 *
 * @Retention(RetentionPolicy.RUNTIME)
 * 表示该注解会在class 字节码文件中存在，在运行时可以通过反射获取到
 *
 * @Documented
 * 标记注解，表示可以生成文档
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyController {

    /**
     * public class MyController
     * 把 class 替换成 @interface 该类即成为注解类
     */

    /**
     * 为Controller 注册别名
     * @return
     */
    String value() default "";

}
```

#### 创建自定义的RequestMapping注解

```java
package com.tjt.springmvc;


import java.lang.annotation.*;


/**
 * @MyRequestMapping 自定义注解类
 *
 * @Target({ElementType.METHOD,ElementType.TYPE})
 * 表示该注解可以作用在方法、类上;
 *
 * @Retention(RetentionPolicy.RUNTIME)
 * 表示该注解会在class 字节码文件中存在，在运行时可以通过反射获取到
 *
 * @Documented
 * 标记注解，表示可以生成文档
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyRequestMapping {

    /**
     * public @interface MyRequestMapping
     * 把 class 替换成 @interface 该类即成为注解类
     */

    /**
     * 表示访问该方法的url
     * @return
     */
    String value() default "";

}
```

#### 设计用于获取项目工程下所有的class 文件的封装工具类

```java
package com.tjt.springmvc;


import java.io.File;
import java.io.FileFilter;
import java.net.JarURLConnection;
import java.net.URL;
import java.net.URLDecoder;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

/**
 * 从项目工程包package 中获取所有的Class 工具类
 */
public class ClassUtils {

    /**
     * 静态常量
     */
    private static String FILE_CONSTANT = "file";
    private static String UTF8_CONSTANT = "UTF-8";
    private static String JAR_CONSTANT = "jar";
    private static String POINT_CLASS_CONSTANT = ".class";
    private static char POINT_CONSTANT = '.';
    private static char LEFT_LINE_CONSTANT = '/';


    /**
     * 定义私有构造函数来屏蔽隐式公有构造函数
     */
    private ClassUtils() {
    }


    /**
     * 从项目工程包package 中获取所有的Class
     * getClasses
     *
     * @param packageName
     * @return
     */
    public static List<Class<?>> getClasses(String packageName) throws Exception {


        List<Class<?>> classes = new ArrayList<Class<?>>();  // 定义一个class 类的泛型集合
        boolean recursive = true;  // recursive 是否循环迭代
        String packageDirName = packageName.replace(POINT_CONSTANT, LEFT_LINE_CONSTANT);  // 获取包的名字 并进行替换
        Enumeration<URL> dirs;  // 定义一个枚举的集合 分别保存该目录下的所有java 类文件及Jar 包等内容
        dirs = Thread.currentThread().getContextClassLoader().getResources(packageDirName);
        /**
         * 循环迭代 处理这个目录下的things
         */
        while (dirs.hasMoreElements()) {
            URL url = dirs.nextElement();  // 获取下一个元素
            String protocol = url.getProtocol();  // 得到协议的名称 protocol
            // 如果是
            /**
             * 若protocol 是文件形式
             */
            if (FILE_CONSTANT.equals(protocol)) {
                String filePath = URLDecoder.decode(url.getFile(), UTF8_CONSTANT); // 获取包的物理路径
                findAndAddClassesInPackageByFile(packageName, filePath, recursive, classes); // 以文件的方式扫描整个包下的文件 并添加到集合中
                /**
                 * 若protocol 是jar 包文件
                 */
            } else if (JAR_CONSTANT.equals(protocol)) {
                JarFile jar;  // 定义一个JarFile
                jar = ((JarURLConnection) url.openConnection()).getJarFile();  // 获取jar
                Enumeration<JarEntry> entries = jar.entries();  // 从jar 包中获取枚举类
                /**
                 * 循环迭代从Jar 包中获得的枚举类
                 */
                while (entries.hasMoreElements()) {
                    JarEntry entry = entries.nextElement();  // 获取jar里的一个实体，如目录、META-INF等文件
                    String name = entry.getName();
                    /**
                     * 若实体名是以 / 开头
                     */
                    if (name.charAt(0) == LEFT_LINE_CONSTANT) {
                        name = name.substring(1);  // 获取后面的字符串
                    }
                    // 如果
                    /**
                     * 若实体名前半部分和定义的包名相同
                     */
                    if (name.startsWith(packageDirName)) {
                        int idx = name.lastIndexOf(LEFT_LINE_CONSTANT);
                        /**
                         * 并且实体名以为'/' 结尾
                         * 若其以'/' 结尾则是一个包
                         */
                        if (idx != -1) {
                            packageName = name.substring(0, idx).replace(LEFT_LINE_CONSTANT, POINT_CONSTANT);  // 获取包名 并把'/' 替换成'.'
                        }
                        /**
                         * 若实体是一个包 且可以继续迭代
                         */
                        if ((idx != -1) || recursive) {
                            if (name.endsWith(POINT_CLASS_CONSTANT) && !entry.isDirectory()) {  // 若为.class 文件 且不是目录
                                String className = name.substring(packageName.length() + 1, name.length() - 6);  // 则去掉.class 后缀并获取真正的类名
                                classes.add(Class.forName(packageName + '.' + className)); // 把获得到的类名添加到classes
                            }
                        }
                    }
                }
            }
        }

        return classes;
    }


    /**
     * 以文件的形式来获取包下的所有Class
     * findAndAddClassesInPackageByFile
     *
     * @param packageName
     * @param packagePath
     * @param recursive
     * @param classes
     */
    public static void findAndAddClassesInPackageByFile(
            String packageName, String packagePath,
            final boolean recursive,
            List<Class<?>> classes) throws Exception {


        File dir = new File(packagePath);  // 获取此包的目录并建立一个File

        if (!dir.exists() || !dir.isDirectory()) {  // 若dir 不存在或者 也不是目录就直接返回
            return;
        }

        File[] dirfiles = dir.listFiles(new FileFilter() {  // 若dir 存在 则获取包下的所有文件、目录

            /**
             * 自定义过滤规则 如果可以循环(包含子目录) 或则是以.class 结尾的文件(编译好的java 字节码文件)
             * @param file
             * @return
             */
            @Override
            public boolean accept(File file) {
                return (recursive && file.isDirectory()) || (file.getName().endsWith(POINT_CLASS_CONSTANT));
            }
        });

        /**
         * 循环所有文件获取java 类文件并添加到集合中
         */
        for (File file : dirfiles) {
            if (file.isDirectory()) {  // 若file 为目录 则继续扫描
                findAndAddClassesInPackageByFile(packageName + "." + file.getName(), file.getAbsolutePath(), recursive,
                        classes);
            } else {  // 若file 为java 类文件 则去掉后面的.class 只留下类名
                String className = file.getName().substring(0, file.getName().length() - 6);
                classes.add(Class.forName(packageName + '.' + className));  // 把className 添加到集合中去

            }
        }
    }
}
```

#### 访问跳转页面`index.jsp`

```html
<%--
  Created by IntelliJ IDEA.
  User: apple
  Date: 2019-11-07
  Time: 13:28
  To change this template use File | Settings | File Templates.
--%>
<%--
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
--%>
<html>
  <head>
    <title>My Fucking SpringMVC</title>
  </head>
  <body>
  <h2>The Lie We Live!</h2>
  <H2>My Fucking SpringMVC</H2>
  </body>
</html>
```

#### 自定义`DispatcherServlet `设计，继承`HttpServlet`，重写`init `、`doGet`、`doPost `等方法，以及自定义注解要实现的功能。

```java
package com.tjt.springmvc;


import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;



/**
 * DispatcherServlet 处理SpringMVC 框架流程
 * 主要流程：
 * 1、包扫描获取包下面所有的类
 * 2、初始化包下面所有的类
 * 3、初始化HandlerMapping 方法,将url 和方法对应上
 * 4、实现HttpServlet 重写doPost 方法
 *
 */
public class DispatcherServlet extends HttpServlet {

    /**
     * 部分静态常量
     */
    private static String PACKAGE_CLASS_NULL_EX = "包扫描后的classes为null";
    private static String HTTP_NOT_EXIST = "sorry http is not exit 404";
    private static String METHOD_NOT_EXIST = "sorry method is not exit 404";
    private static String POINT_JSP = ".jsp";
    private static String LEFT_LINE = "/";

    /**
     * 用于存放SpringMVC bean 的容器
     */
    private ConcurrentHashMap<String, Object> mvcBeans = new ConcurrentHashMap<>();
    private ConcurrentHashMap<String, Object> mvcBeanUrl = new ConcurrentHashMap<>();
    private ConcurrentHashMap<String, String> mvcMethodUrl = new ConcurrentHashMap<>();
    private static String PROJECT_PACKAGE_PATH = "com.tjt.springmvc";


    /**
     * 按顺序初始化组件
     * @param config
     */
    @Override
    public void init(ServletConfig config) {
        String packagePath = PROJECT_PACKAGE_PATH;
        try {
            //1.进行报扫描获取当前包下面所有的类
            List<Class<?>> classes = comscanPackage(packagePath);
            //2.初始化springmvcbean
            initSpringMvcBean(classes);
        } catch (Exception e) {
            e.printStackTrace();
        }
        //3.将请求地址和方法进行映射
        initHandMapping(mvcBeans);
    }


    /**
     * 调用ClassUtils 工具类获取工程中所有的class
     * @param packagePath
     * @return
     * @throws Exception
     */
    public List<Class<?>> comscanPackage(String packagePath) throws Exception {
        List<Class<?>> classes = ClassUtils.getClasses(packagePath);
        return classes;
    }

    /**
     * 初始化SpringMVC bean
     *
     * @param classes
     * @throws Exception
     */
    public void initSpringMvcBean(List<Class<?>> classes) throws Exception {
        /**
         * 若包扫描出的classes 为空则直接抛异常
         */
        if (classes.isEmpty()) {
            throw new Exception(PACKAGE_CLASS_NULL_EX);
        }

        /**
         * 遍历所有classes 获取@MyController 注解
         */
        for (Class<?> aClass : classes) {
            //获取被自定义注解的controller 将其初始化到自定义springmvc 容器中
            MyController declaredAnnotation = aClass.getDeclaredAnnotation(MyController.class);
            if (declaredAnnotation != null) {
                //获取类的名字
                String beanid = lowerFirstCapse(aClass.getSimpleName());
                //获取对象
                Object beanObj = aClass.newInstance();
                //放入spring 容器
                mvcBeans.put(beanid, beanObj);
            }
        }

    }

    /**
     * 初始化HandlerMapping 方法
     *
     * @param mvcBeans
     */
    public void initHandMapping(ConcurrentHashMap<String, Object> mvcBeans) {
        /**
         * 遍历springmvc 获取注入的对象值
         */
        for (Map.Entry<String, Object> entry : mvcBeans.entrySet()) {
            Object objValue = entry.getValue();
            Class<?> aClass = objValue.getClass();
            //获取当前类 判断其是否有自定义的requestMapping 注解
            String mappingUrl = null;
            MyRequestMapping anRequestMapping = aClass.getDeclaredAnnotation(MyRequestMapping.class);
            if (anRequestMapping != null) {
                mappingUrl = anRequestMapping.value();
            }
            //获取当前类所有方法,判断方法上是否有注解
            Method[] declaredMethods = aClass.getDeclaredMethods();
            /**
             * 遍历注解
             */
            for (Method method : declaredMethods) {
                MyRequestMapping methodDeclaredAnnotation = method.getDeclaredAnnotation(MyRequestMapping.class);
                if (methodDeclaredAnnotation != null) {
                    String methodUrl = methodDeclaredAnnotation.value();
                    mvcBeanUrl.put(mappingUrl + methodUrl, objValue);
                    mvcMethodUrl.put(mappingUrl + methodUrl, method.getName());
                }
            }

        }

    }

    /**
     * @param str
     * @return 类名首字母小写
     */
    public static String lowerFirstCapse(String str) {
        char[] chars = str.toCharArray();
        chars[0] += 32;
        return String.valueOf(chars);

    }

    /**
     * doPost 请求
     * @param req
     * @param resp
     * @throws ServletException
     * @throws IOException
     */
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try {
            /**
             * 处理请求
             */
            doServelt(req, resp);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * doServelt 处理请求
     * @param req
     * @param resp
     * @throws IOException
     * @throws NoSuchMethodException
     * @throws InvocationTargetException
     * @throws IllegalAccessException
     * @throws ServletException
     */
    private void doServelt(HttpServletRequest req, HttpServletResponse resp) throws IOException, NoSuchMethodException, InvocationTargetException, IllegalAccessException, ServletException {
        //获取请求地址
        String requestUrl = req.getRequestURI();
        //查找地址所对应bean
        Object object = mvcBeanUrl.get(requestUrl);
        if (Objects.isNull(object)) {
            resp.getWriter().println(HTTP_NOT_EXIST);
            return;
        }
        //获取请求的方法
        String methodName = mvcMethodUrl.get(requestUrl);
        if (methodName == null) {
            resp.getWriter().println(METHOD_NOT_EXIST);
            return;
        }


        //通过构反射执行方法
        Class<?> aClass = object.getClass();
        Method method = aClass.getMethod(methodName);

        String invoke = (String) method.invoke(object);
        // 获取后缀信息
        String suffix = POINT_JSP;
        // 页面目录地址
        String prefix = LEFT_LINE;
        req.getRequestDispatcher(prefix + invoke + suffix).forward(req, resp);




    }

    /**
     * doGet 请求
     * @param req
     * @param resp
     * @throws ServletException
     * @throws IOException
     */
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        this.doPost(req, resp);
    }


}
```

#### 测试手写SpringMVC 框架效果类TestMySpringMVC 

```java
package com.tjt.springmvc;


/**
 * 手写SpringMVC 测试类
 * TestMySpringMVC
 */
@MyController
@MyRequestMapping(value = "/tjt")
public class TestMySpringMVC {


    /**
     * 测试手写SpringMVC 框架效果 testMyMVC1
     * @return
     */
    @MyRequestMapping("/mvc")
    public String testMyMVC1() {
        System.out.println("he Lie We Live!");
        return "index";
    }


}
```

#### 配置Tomcat 用于运行Web 项目

![配置Tomcat](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109233457845-1004198702.png) 

#### 运行项目，访问测试

1、输入正常路径` http://localhost:8080/tjt/mvc `访问测试效果如下： 

![正常路径测试效果](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109234051593-1526388414.png) 

2、输入非法(不存在)路径`http://localhost:8080/tjt/mvc8` 访问测试效果如下： 

![非法路径请求效果](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109234133072-507532943.png) 

3、控制台打印“`The Lie We Live`”如下： 

![控制台输出](https://img2018.cnblogs.com/common/1580332/201911/1580332-20191109234405430-1339702138.png) 

测试效果如上则证明手写SpringMVC 框架 已成功。

好了，今天分享到这里。

## 总结

本文知识按照`Spring MVC`的大致思想，写了一个简单版的，如果感兴趣可以把这个项目继续完善。















