面试官：Spring 注解 @After、@Around、@Before 的执行顺序是？

在AOP编程中，有`@Before`，`@After`，`@Around`，`@AfterRunning`注解等等。

首先上下自己的代码，定义了切点的定义

```
@Aspect
@Component
public class LogApsect {
 
    private static final Logger logger = LoggerFactory.getLogger(LogApsect.class);
 
    ThreadLocal<Long> startTime = new ThreadLocal<>();
 
    // 第一个*代表返回类型不限
    // 第二个*代表所有类
    // 第三个*代表所有方法
    // (..) 代表参数不限
    @Pointcut("execution(public * com.lmx.blog.controller.*.*(..))")
    @Order(2)
    public void pointCut(){};
 
    @Pointcut("@annotation(com.lmx.blog.annotation.RedisCache)")
    @Order(1) // Order 代表优先级，数字越小优先级越高
    public void annoationPoint(){};
 
    @Before(value = "annoationPoint() || pointCut()")
    public void before(JoinPoint joinPoint){
        System.out.println("方法执行前执行......before");
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        logger.info("<=====================================================");
        logger.info("请求来源： =》" + request.getRemoteAddr());
        logger.info("请求URL：" + request.getRequestURL().toString());
        logger.info("请求方式：" + request.getMethod());
        logger.info("响应方法：" + joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
        logger.info("请求参数：" + Arrays.toString(joinPoint.getArgs()));
        logger.info("------------------------------------------------------");
        startTime.set(System.currentTimeMillis());
    }
 
    // 定义需要匹配的切点表达式，同时需要匹配参数
    @Around("pointCut() && args(arg)")
    public Response around(ProceedingJoinPoint pjp,String arg) throws Throwable{
        System.out.println("name:" + arg);
        System.out.println("方法环绕start...around");
        String result = null;
        try{
            result = pjp.proceed().toString() + "aop String";
            System.out.println(result);
        }catch (Throwable e){
            e.printStackTrace();
        }
        System.out.println("方法环绕end...around");
        return (Response) pjp.proceed();
    }
 
    @After("within(com.lmx.blog.controller.*Controller)")
    public void after(){
        System.out.println("方法之后执行...after.");
    }
 
    @AfterReturning(pointcut="pointCut()",returning = "rst")
    public void afterRunning(Response rst){
        if(startTime.get() == null){
            startTime.set(System.currentTimeMillis());
        }
        System.out.println("方法执行完执行...afterRunning");
        logger.info("耗时（毫秒）：" +  (System.currentTimeMillis() - startTime.get()));
        logger.info("返回数据：{}", rst);
        logger.info("==========================================>");
    }
 
    @AfterThrowing("within(com.lmx.blog.controller.*Controller)")
    public void afterThrowing(){
        System.out.println("异常出现之后...afterThrowing");
    }
 
 
}
```

`@Before`，`@After`，`@Around`注解的区别大家可以自行百度下。

总之就是`@Around`可以实现`@Before`和`@After`的功能，并且只需要在一个方法中就可以实现。推荐一个 Spring Boot 基础教程及实战示例：https://github.com/javastacks/spring-boot-best-practice

首先我们来测试一个方法用于获取数据库一条记录的

```
@RequestMapping("/achieve")
public Response achieve(){
    System.out.println("方法执行-----------");
    return Response.ok(articleDetailSercice.getPrimaryKeyById(1L));
}
```

以下是控制台打印的日志

```
方法执行前执行......before
2018-11-23 16:31:59.795 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - <=====================================================
2018-11-23 16:31:59.795 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 请求来源： =》0:0:0:0:0:0:0:1
2018-11-23 16:31:59.795 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 请求URL：http://localhost:8888/user/achieve
2018-11-23 16:31:59.795 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 请求方式：GET
2018-11-23 16:31:59.795 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 响应方法：com.lmx.blog.controller.UserController.achieve
2018-11-23 16:31:59.796 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 请求参数：[]
2018-11-23 16:31:59.796 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - ------------------------------------------------------
方法执行-----------
2018-11-23 16:31:59.806 [http-nio-8888-exec-9] DEBUG c.l.b.m.A.selectPrimaryKey - ==>  Preparing: select * from article_detail where id = ? 
2018-11-23 16:31:59.806 [http-nio-8888-exec-9] DEBUG c.l.b.m.A.selectPrimaryKey - ==>  Preparing: select * from article_detail where id = ? 
2018-11-23 16:31:59.806 [http-nio-8888-exec-9] DEBUG c.l.b.m.A.selectPrimaryKey - ==> Parameters: 1(Long)
2018-11-23 16:31:59.806 [http-nio-8888-exec-9] DEBUG c.l.b.m.A.selectPrimaryKey - ==> Parameters: 1(Long)
2018-11-23 16:31:59.814 [http-nio-8888-exec-9] DEBUG c.l.b.m.A.selectPrimaryKey - <==      Total: 1
2018-11-23 16:31:59.814 [http-nio-8888-exec-9] DEBUG c.l.b.m.A.selectPrimaryKey - <==      Total: 1
方法之后执行...after.
方法执行完执行...afterRunning
2018-11-23 16:31:59.824 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 耗时（毫秒）：27
2018-11-23 16:31:59.824 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - 返回数据：com.lmx.blog.common.Response@8675ce5
2018-11-23 16:31:59.824 [http-nio-8888-exec-9] INFO  c.l.blog.config.LogApsect - ==========================================>
```

可以看到，因为没有匹配`@Around`的规则，所以没有进行环绕通知。（PS：我定义的环绕通知意思是要符合是 controller 包下的方法并且方法必须带有参数，而上述方法没有参数，所以只走了`@before`和`@after`方法，不符合`@Around`的匹配逻辑）

我们再试一下另一个带有参数的方法

```
@RedisCache(type = Response.class)
@RequestMapping("/sendEmail")
public Response sendEmailToAuthor(String content){
    System.out.println("测试执行次数");
    return Response.ok(true);
}
```

以下是该部分代码的console打印

```
name:第二封邮件呢
方法环绕start...around
方法执行前执行......before
2018-11-23 16:34:55.347 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - <=====================================================
2018-11-23 16:34:55.347 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求来源： =》0:0:0:0:0:0:0:1
2018-11-23 16:34:55.347 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求URL：http://localhost:8888/user/sendEmail
2018-11-23 16:34:55.348 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求方式：GET
2018-11-23 16:34:55.348 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 响应方法：com.lmx.blog.controller.UserController.sendEmailToAuthor
2018-11-23 16:34:55.348 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求参数：[第二封邮件呢]
2018-11-23 16:34:55.348 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - ------------------------------------------------------
测试执行次数
com.lmx.blog.common.Response@6d17f2fdaop String
方法环绕end...around
方法执行前执行......before
2018-11-23 16:34:55.349 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - <=====================================================
2018-11-23 16:34:55.349 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求来源： =》0:0:0:0:0:0:0:1
2018-11-23 16:34:55.349 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求URL：http://localhost:8888/user/sendEmail
2018-11-23 16:34:55.349 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求方式：GET
2018-11-23 16:34:55.349 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 响应方法：com.lmx.blog.controller.UserController.sendEmailToAuthor
2018-11-23 16:34:55.349 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 请求参数：[第二封邮件呢]
2018-11-23 16:34:55.350 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - ------------------------------------------------------
测试执行次数
方法之后执行...after.
方法执行完执行...afterRunning
2018-11-23 16:34:55.350 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 耗时（毫秒）：0
2018-11-23 16:34:55.350 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - 返回数据：com.lmx.blog.common.Response@79f85428
2018-11-23 16:34:55.350 [http-nio-8888-exec-2] INFO  c.l.blog.config.LogApsect - ==========================================>
```

显而易见，该方法符合 `@Around` 环绕通知的匹配规则，所以进入了`@Around`的逻辑，但是发现了问题，所有的方法都被执行了2次，不管是切面层还是方法层。（有人估计要问我不是用的自定义注解 `@RedisCache(type = Response.class)` 么。为什么会符合 `@Around`的匹配规则呢，这个等会在下面说）

我们分析日志的打印顺序可以得出，在执行环绕方法时候，会优先进入 `@Around`下的方法。`@Around`的方法再贴一下代码。

```
// 定义需要匹配的切点表达式，同时需要匹配参数
@Around("pointCut() && args(arg)")
public Response around(ProceedingJoinPoint pjp,String arg) throws Throwable{
    System.out.println("name:" + arg);
    System.out.println("方法环绕start...around");
    String result = null;
    try{
        result = pjp.proceed().toString() + "aop String";
        System.out.println(result);
    }catch (Throwable e){
        e.printStackTrace();
    }
    System.out.println("方法环绕end...around");
    return (Response) pjp.proceed();
}
```

打印了前两行代码以后，转而去执行了 `@Before`方法，是因为中途触发了  `ProceedingJoinPoint.proceed()` 方法。

这个方法的作用是执行被代理的方法，也就是说执行了这个方法之后会执行我们controller的方法，而后执行 `@before`，`@after`，然后回到@Around执行未执行的方法，最后执行 `@afterRunning`，如果有异常抛出能执行 `@AfterThrowing`

也就是说环绕的执行顺序是   `@Around→@Before→@After→@Around`执行 `ProceedingJoinPoint.proceed()` 之后的操作→`@AfterRunning`(如果有异常→`@AfterThrowing`)

而我们上述的日志相当于把上述结果执行了2遍，根本原因在于 `ProceedingJoinPoint.proceed()` 这个方法，可以发现在@Around 方法中我们使用了2次这个方法，然而每次调用这个方法时都会走一次`@Before→@After→@Around`执行 `ProceedingJoinPoint.proceed()` 之后的操作→`@AfterRunning`(如果有异常→`@AfterThrowing`)。

因此问题是出现在这里。所以更改`@Around`部分的代码即可解决该问题。更改之后的代码如下：

```
@Around("pointCut() && args(arg)")
public Response around(ProceedingJoinPoint pjp,String arg) throws Throwable{
    System.out.println("name:" + arg);
    System.out.println("方法环绕start...around");
    String result = null;
    Object object = pjp.proceed();
    try{
        result = object.toString() + "aop String";
        System.out.println(result);
    }catch (Throwable e){
        e.printStackTrace();
    }
    System.out.println("方法环绕end...around");
    return (Response) object;
}
```

更改代码之后的运行结果

```
name:第二封邮件呢
方法环绕start...around
方法执行前执行......before
2018-11-23 16:52:14.315 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - <=====================================================
2018-11-23 16:52:14.315 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 请求来源： =》0:0:0:0:0:0:0:1
2018-11-23 16:52:14.315 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 请求URL：http://localhost:8888/user/sendEmail
2018-11-23 16:52:14.315 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 请求方式：GET
2018-11-23 16:52:14.316 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 响应方法：com.lmx.blog.controller.UserController.sendEmailToAuthor
2018-11-23 16:52:14.316 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 请求参数：[第二封邮件呢]
2018-11-23 16:52:14.316 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - ------------------------------------------------------
测试执行次数
com.lmx.blog.common.Response@1b1c76afaop String
方法环绕end...around
方法之后执行...after.
方法执行完执行...afterRunning
2018-11-23 16:52:14.316 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 耗时（毫秒）：0
2018-11-23 16:52:14.316 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - 返回数据：com.lmx.blog.common.Response@1b1c76af
2018-11-23 16:52:14.316 [http-nio-8888-exec-4] INFO  c.l.blog.config.LogApsect - ==========================================>
```

**回到上述未解决的问题，为什么我定义了切面的另一个注解还可以进入@Around方法呢？**

因为我们的方法仍然在controller下，因此满足该需求，如果我们定义了controller包下的某个controller才有用。

例如：

```
@Pointcut("execution(public * com.lmx.blog.controller.UserController.*(..))")
```

而如果我们刚才定义的方法是写在 TestController 之下的，那么就不符合 `@Around`方法的匹配规则了，也不符合`@before`和`@after`的注解规则，因此不会匹配任何一个规则，如果需要匹配特定的方法，可以用自定义的注解形式或者特性controller下的方法

①：特性的注解形式

```
@Pointcut("@annotation(com.lmx.blog.annotation.RedisCache)")
@Order(1) // Order 代表优先级，数字越小优先级越高
public void annoationPoint(){};
```

然后在所需要的方法上加入 `@RedisCache`注解，在`@Before`，`@After`，`@Around`等方法上添加该切点的方法名（“`annoationPoint()`”），如果有多个注解需要匹配则用 `||` 隔开

②：指定controller或者指定controller下的方法

```
@Pointcut("execution(public * com.lmx.blog.controller.UserController.*(..))")
@Order(2)
public void pointCut(){};
```

该部分代码是指定了 `com.lmx.blog.controller`包下的UserController下的所有方法。另外，Spring 系列面试题和答案全部整理好了，微信搜索Java技术栈，在后台发送：面试，可以在线阅读。

第一个`*`代表的是返回类型不限

第二个`*`代表的是该controller下的所有方法，`(..)`代表的是参数不限

##### 总结

当方法符合切点规则不符合环绕通知的规则时候，执行的顺序如下

**@Before→@After→@AfterRunning(如果有异常→@AfterThrowing)**

当方法符合切点规则并且符合环绕通知的规则时候，执行的顺序如下

**@Around→@Before→@Around→@After执行 ProceedingJoinPoint.proceed() 之后的操作→@AfterRunning(如果有异常→@AfterThrowing)**

阿达的