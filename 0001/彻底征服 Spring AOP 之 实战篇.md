彻底征服 Spring AOP 之 实战篇

## Spring AOP 实战

看了上面这么多的理论知识, 不知道大家有没有觉得枯燥哈. 不过不要急, 俗话说理论是实践的基础, 对 Spring AOP 有了基本的理论认识后, 我们来看一下下面几个具体的例子吧.
下面的几个例子是我在工作中所遇见的比较常用的 Spring AOP 的使用场景, 我精简了很多有干扰我们学习的注意力的细枝末节, 以力求整个例子的简洁性.

下面几个 Demo 的源码都可以在我的 [Github](https://github.com/yongshun/some_java_code) 上下载到.

### HTTP 接口鉴权

首先让我们来想象一下如下场景: 我们需要提供的 HTTP RESTful 服务, 这个服务会提供一些比较敏感的信息, 因此对于某些接口的调用会进行调用方权限的校验, 而某些不太敏感的接口则不设置权限, 或所需要的权限比较低(例如某些监控接口, 服务状态接口等).
实现这样的需求的方法有很多, 例如我们可以在每个 HTTP 接口方法中对服务请求的调用方进行权限的检查, 当调用方权限不符时, 方法返回错误. 当然这样做并无不可, 不过如果我们的 api 接口很多, 每个接口都进行这样的判断, 无疑有很多冗余的代码, 并且很有可能有某个粗心的家伙忘记了对调用者的权限进行验证, 这样就会造成潜在的 bug.
那么除了上面的所说的方法外, 还有没有别的比较优雅的方式来实现呢? 当然有啦, 不然我在这啰嗦半天干嘛呢, 它就是我们今天的主角: `AOP`.

让我们来提炼一下我们的需求:

1. 可以定制地为某些指定的 HTTP RESTful api 提供权限验证功能.
2. 当调用方的权限不符时, 返回错误.

根据上面所提出的需求, 我们可以进行如下设计:

1. 提供一个特殊的注解 `AuthChecker`, 这个是一个方法注解, 有此注解所标注的 Controller 需要进行调用方权限的认证.
2. 利用 Spring AOP, 以 **@annotation** 切点标志符来匹配有注解 `AuthChecker` 所标注的 joinpoint.
3. 在 advice 中, 简单地检查调用者请求中的 Cookie 中是否有我们指定的 token, 如果有, 则认为此调用者权限合法, 允许调用, 反之权限不合法, 范围错误.

根据上面的设计, 我们来看一下具体的源码吧.
首先是 `AuthChecker` 注解的定义:
**AuthChecker.java:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthChecker {
}
```

`AuthChecker` 注解是一个方法注解, 它用于注解 RequestMapping 方法.

有了注解的定义, 那我们再来看一下 aspect 的实现吧:
**HttpAopAdviseDefine.java:**

```java
@Component
@Aspect
public class HttpAopAdviseDefine {

    // 定义一个 Pointcut, 使用 切点表达式函数 来描述对哪些 Join point 使用 advise.
    @Pointcut("@annotation(com.xys.demo1.AuthChecker)")
    public void pointcut() {
    }

    // 定义 advise
    @Around("pointcut()")
    public Object checkAuth(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes())
                .getRequest();

        // 检查用户所传递的 token 是否合法
        String token = getUserToken(request);
        if (!token.equalsIgnoreCase("123456")) {
            return "错误, 权限不合法!";
        }

        return joinPoint.proceed();
    }

    private String getUserToken(HttpServletRequest request) {
        Cookie[] cookies = request.getCookies();
        if (cookies == null) {
            return "";
        }
        for (Cookie cookie : cookies) {
            if (cookie.getName().equalsIgnoreCase("user_token")) {
                return cookie.getValue();
            }
        }
        return "";
    }
}
```

在这个 aspect 中, 我们首先定义了一个 pointcut, 以 **@annotation** 切点标志符来匹配有注解 `AuthChecker` 所标注的 joinpoint, 即:

```java
// 定义一个 Pointcut, 使用 切点表达式函数 来描述对哪些 Join point 使用 advise.
@Pointcut("@annotation(com.xys.demo1.AuthChecker)")
public void pointcut() {
}
```

然后再定义一个 advice:

```java
// 定义 advise
@Around("pointcut()")
public Object checkAuth(ProceedingJoinPoint joinPoint) throws Throwable {
    HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes())
            .getRequest();

    // 检查用户所传递的 token 是否合法
    String token = getUserToken(request);
    if (!token.equalsIgnoreCase("123456")) {
        return "错误, 权限不合法!";
    }

    return joinPoint.proceed();
}
```

当被 `AuthChecker` 注解所标注的方法调用前, 会执行我们的这个 advice, 而这个 advice 的处理逻辑很简单, 即从 HTTP 请求中获取名为 `user_token` 的 cookie 的值, 如果它的值是 `123456`, 则我们认为此 HTTP 请求合法, 进而调用 `joinPoint.proceed()` 将 HTTP 请求转交给相应的控制器处理; 而如果`user_token` cookie 的值不是 `123456`, 或为空, 则认为此 HTTP 请求非法, 返回错误.

接下来我们来写一个模拟的 HTTP 接口:
**DemoController.java:**

```java
@RestController
public class DemoController {
    @RequestMapping("/aop/http/alive")
    public String alive() {
        return "服务一切正常";
    }

    @AuthChecker
    @RequestMapping("/aop/http/user_info")
    public String callSomeInterface() {
        return "调用了 user_info 接口.";
    }
}
```

注意到上面我们提供了两个 HTTP 接口, 其中 接口 **/aop/http/alive** 是没有 `AuthChecker` 标注的, 而 **/aop/http/user_info** 接口则用到了 `@AuthChecker` 标注. 那么自然地, 当请求了 **/aop/http/user_info** 接口时, 就会触发我们所设置的权限校验逻辑.

接下来我们来验证一下, 我们所实现的功能是否有效吧.
首先在 Postman 中, 调用 **/aop/http/alive** 接口, 请求头中不加任何参数:

![1622683754614](E:\other\网络\assets\1622683754614.png)

可以看到, 我们的 HTTP 请求完全没问题.

那么再来看一下请求 **/aop/http/user_info** 接口会怎样呢:

![1622683653350](E:\other\网络\assets\1622683653350.png)

当我们请求 **/aop/http/user_info** 接口时, 服务返回一个权限异常的错误, 为什么会这样呢? 自然就是我们的权限认证系统起了作为: 当一个方法被调用并且这个方法有 `AuthChecker` 标注时, 那么首先会执行到我们的 `around advice`, 在这个 advice 中, 我们会校验 HTTP 请求的 cookie 字段中是否有携带 `user_token` 字段时, 如果没有, 则返回权限错误.
那么为了能够正常地调用 **/aop/http/user_info** 接口, 我们可以在 Cookie 中添加 **user_token=123456**, 这样我们可以愉快的玩耍了:

![1622683643407](E:\other\网络\assets\1622683643407.png)

> `注意`, Postman 默认是不支持 Cookie 的, 所以为了实现添加 Cookie 的功能, 我们需要安装 Postman 的 `interceptor` 插件. 安装方法可以看[官网的文章](https://www.getpostman.com/docs/interceptor_cookies)

#### 完整源码

[HTTP 接口鉴权](https://github.com/yongshun/some_java_code/tree/master/SpringAOPDemo/src/main/java/com/xys/demo1)

### 方法调用日志

第二个 AOP 实例是记录一个方法调用的log. 这应该是一个很常见的功能了.
首先假设我们有如下需求:

1. 某个服务下的方法的调用需要有 log: 记录调用的参数以及返回结果.
2. 当方法调用出异常时, 有特殊处理, 例如打印异常 log, 报警等.

根据上面的需求, 我们可以使用 before advice 来在调用方法前打印调用的参数, 使用 after returning advice 在方法返回打印返回的结果. 而当方法调用失败后, 可以使用 after throwing advice 来做相应的处理.
那么我们来看一下 aspect 的实现:

```java
@Component
@Aspect
public class LogAopAdviseDefine {
    private Logger logger = LoggerFactory.getLogger(getClass());

    // 定义一个 Pointcut, 使用 切点表达式函数 来描述对哪些 Join point 使用 advise.
    @Pointcut("within(NeedLogService)")
    public void pointcut() {
    }

    // 定义 advise
    @Before("pointcut()")
    public void logMethodInvokeParam(JoinPoint joinPoint) {
        logger.info("---Before method {} invoke, param: {}---", joinPoint.getSignature().toShortString(), joinPoint.getArgs());
    }

    @AfterReturning(pointcut = "pointcut()", returning = "retVal")
    public void logMethodInvokeResult(JoinPoint joinPoint, Object retVal) {
        logger.info("---After method {} invoke, result: {}---", joinPoint.getSignature().toShortString(), joinPoint.getArgs());
    }

    @AfterThrowing(pointcut = "pointcut()", throwing = "exception")
    public void logMethodInvokeException(JoinPoint joinPoint, Exception exception) {
        logger.info("---method {} invoke exception: {}---", joinPoint.getSignature().toShortString(), exception.getMessage());
    }
}
```

第一步, 自然是定义一个 `pointcut`, 以 **within** 切点标志符来匹配类 `NeedLogService` 下的所有 joinpoint, 即:

```java
@Pointcut("within(NeedLogService)")
public void pointcut() {
}
```

接下来根据我们前面的设计, 我们分别定义了三个 advice, 第一个是一个 before advice:

```java
@Before("pointcut()")
public void logMethodInvokeParam(JoinPoint joinPoint) {
    logger.info("---Before method {} invoke, param: {}---", joinPoint.getSignature().toShortString(), joinPoint.getArgs());
}
```

它在一个符合要求的 joinpoint 方法调用前执行, 打印调用的方法名和调用的参数.

第二个是 after return advice:

```java
@AfterReturning(pointcut = "pointcut()", returning = "retVal")
public void logMethodInvokeResult(JoinPoint joinPoint, Object retVal) {
    logger.info("---After method {} invoke, result: {}---", joinPoint.getSignature().toShortString(), joinPoint.getArgs());
}
```

这个 advice 会在方法调用成功后打印出方法名还反的参数.

最后一个是 after throw advice:

```java
@AfterThrowing(pointcut = "pointcut()", throwing = "exception")
public void logMethodInvokeException(JoinPoint joinPoint, Exception exception) {
    logger.info("---method {} invoke exception: {}---", joinPoint.getSignature().toShortString(), exception.getMessage());
}
```

这个 advice 会在指定的 joinpoint 抛出异常时执行, 打印异常的信息.

接下来我们再写两个 Service 类:
**NeedLogService.java:**

```java
@Service
public class NeedLogService {
    private Logger logger = LoggerFactory.getLogger(getClass());
    private Random random = new Random(System.currentTimeMillis());

    public int logMethod(String someParam) {
        logger.info("---NeedLogService: logMethod invoked, param: {}---", someParam);
        return random.nextInt();
    }

    public void exceptionMethod() throws Exception {
        logger.info("---NeedLogService: exceptionMethod invoked---");
        throw new Exception("Something bad happened!");
    }
}
```

**NormalService.java:**

```java
@Service
public class NormalService {
    private Logger logger = LoggerFactory.getLogger(getClass());

    public void someMethod() {
        logger.info("---NormalService: someMethod invoked---");
    }
}
```

根据我们 pointcut 的规则, 类 NeedLogService 下的所有方法都会被织入 advice, 而类 NormalService 则不会.

最后我们分别调用这几个方法:

```java
@PostConstruct
public void test() {
    needLogService.logMethod("xys");
    try {
        needLogService.exceptionMethod();
    } catch (Exception e) {
        // Ignore
    }
    normalService.someMethod();
}
```

我们可以看到有如下输出:

```java
---Before method NeedLogService.logMethod(..) invoke, param: [xys]---
---NeedLogService: logMethod invoked, param: xys---
---After method NeedLogService.logMethod(..) invoke, result: [xys]---

---Before method NeedLogService.exceptionMethod() invoke, param: []---
---NeedLogService: exceptionMethod invoked---
---method NeedLogService.exceptionMethod() invoke exception: Something bad happened!---

---NormalService: someMethod invoked---
```

根据 log, 我们知道, NeedLogService.logMethod 执行的前后确实有 advice 执行了, 并且在 NeedLogService.exceptionMethod 抛出异常后, `logMethodInvokeException` 这个 advice 也被执行了. 而由于 pointcut 的匹配规则, 在 `NormalService` 类中的方法则不会织入 advice.

#### 完整源码

[方法调用日志](https://github.com/yongshun/some_java_code/tree/master/SpringAOPDemo/src/main/java/com/xys/demo2)

### 方法耗时统计

作为程序员, 我们都知道服务监控对于一个服务能够长期稳定运行的重要性, 因此很多公司都有自己内部的监控报警系统, 或者是使用一些开源的系统, 例如小米的 Falcon 监控系统.

那么在程序监控中, AOP 有哪些用武之地呢? 我们来假想一下如下场景:

> 有一天, leader 对小王说, "小王啊, 你负责的那个服务不太稳定啊, 经常有超时发生! 你有对这些服务接口进行过耗时统计吗?"
> 耗时统计? 小王嘀咕了, 小声的回答到: "还没有加呢."
> leader: "你看着办吧, 我明天要看到各个时段的服务接口调用的耗时分布!"
> 小王这就犯难了, 虽然说计算一个方法的调用耗时并不是一个很难的事情, 但是整个服务有二十来个接口呢, 一个一个地添加统计代码, 那还不是要累死人了.
> 看着同事一个一个都下班回家了, 小王眉头更加紧了. 不过此时小王灵机一动: "噫, 有了!".
> 小王想到了一个好方法, 立即动手, 吭哧吭哧地几分钟就搞定了.

那么小王的解决方法是什么呢? 自然是我们的主角 `AOP` 啦.

首先让我们来提炼一下需求:

1. 为服务中的每个方法调用进行调用耗时记录.
2. 将方法调用的时间戳, 方法名, 调用耗时上报到监控平台

有了需求, 自然设计实现就很简单了. 首先我们可以使用 around advice, 然后在方法调用前, 记录一下开始时间, 然后在方法调用结束后, 记录结束时间, 它们的时间差就是方法的调用耗时.

我们来看一下具体的 aspect 实现:

**ExpiredAopAdviseDefine.java:**

```java
@Component
@Aspect
public class ExpiredAopAdviseDefine {
    private Logger logger = LoggerFactory.getLogger(getClass());

    // 定义一个 Pointcut, 使用 切点表达式函数 来描述对哪些 Join point 使用 advise.
    @Pointcut("within(SomeService)")
    public void pointcut() {
    }

    // 定义 advise
    // 定义 advise
    @Around("pointcut()")
    public Object methodInvokeExpiredTime(ProceedingJoinPoint pjp) throws Throwable {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        // 开始
        Object retVal = pjp.proceed();
        stopWatch.stop();
        // 结束

        // 上报到公司监控平台
        reportToMonitorSystem(pjp.getSignature().toShortString(), stopWatch.getTotalTimeMillis());

        return retVal;
    }


    public void reportToMonitorSystem(String methodName, long expiredTime) {
        logger.info("---method {} invoked, expired time: {} ms---", methodName, expiredTime);
        //
    }
}
```

aspect 一开始定义了一个 `pointcut`, 匹配 `SomeService` 类下的所有的方法.
接着呢, 定义了一个 around advice:

```java
@Around("pointcut()")
public Object methodInvokeExpiredTime(ProceedingJoinPoint pjp) throws Throwable {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    // 开始
    Object retVal = pjp.proceed();
    stopWatch.stop();
    // 结束

    // 上报到公司监控平台
    reportToMonitorSystem(pjp.getSignature().toShortString(), stopWatch.getTotalTimeMillis());

    return retVal;
}
```

advice 中的代码也很简单, 它使用了 Spring 提供的 StopWatch 来统计一段代码的执行时间. 首先我们先调用 **stopWatch.start()** 开始计时, 然后通过 `pjp.proceed()` 来调用我们实际的服务方法, 当调用结束后, 通过 **stopWatch.stop()** 来结束计时.

接着我们来写一个简单的服务, 这个服务提供一个 **someMethod** 方法用于模拟一个耗时的方法调用:
**SomeService.java:**

```java
@Service
public class SomeService {
    private Logger logger = LoggerFactory.getLogger(getClass());
    private Random random = new Random(System.currentTimeMillis());

    public void someMethod() {
        logger.info("---SomeService: someMethod invoked---");
        try {
            // 模拟耗时任务
            Thread.sleep(random.nextInt(500));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

这样当 `SomeService` 类下的方法调用时, 我们所提供的 advice 就会被执行, 因此就可以自动地为我们统计此方法的调用耗时, 并自动上报到监控系统中了.
看到 `AOP` 的威力了吧, 我们这里仅仅使用了寥寥数语就把一个需求完美地解决了, 并且还与原来的业务逻辑完全解耦, 扩展及其方便.

#### 完整源码

[方法耗时统计](https://github.com/yongshun/some_java_code/tree/master/SpringAOPDemo/src/main/java/com/xys/demo3)

### 总结

通过上面的几个简单例子, 我们对 `Spring AOP` 的使用应该有了一个更为深入的了解了. 其实 Spring AOP 的使用的地方不止这些, 例如 Spring 的 `声明式事务` 就是在 AOP 之上构建的. 读者朋友也可以根据自己的实际业务场景, 合理使用 Spring AOP, 发挥它的强大功能!