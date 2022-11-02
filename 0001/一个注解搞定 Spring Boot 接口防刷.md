美团面试官：接口防刷，如何做到？

大家好，我是老田，今天给大家分享的是：一个注解搞定 Spring Boot 接口防刷。

之前，我已经分享给四个美团面试的技术点：



> 面试官：一个接口如何防刷？
>
> 我：这个没搞过（每天CRUD，真的没搞过）
>
> 面试官：如果现在让你来设计，你会怎么设计？
>
> 我：巴拉巴拉...胡扯一通
>
> 面试官：我们还是换个话题吧
>
> .....

为了不让大家也和我有同样的遭遇，今天，咱们就用一个非常简单的方式实现防刷：

> 一个注解搞定防刷

## 技术点

涉及到的技术点有如下几个：

- 拦截器
- `Redis`的get、set
- Spring Boot项目

其实，非常简单，主要的还是看业务。

本文目录：

![1628257537653](E:\other\网络\assets\1628257537653.png)

## 自定义注解

自顶一个注解，放在方法上，所以Target中注明是Method。

```java
import java.lang.annotation.Retention;
import java.lang.annotation.Target;
 
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
 
@Retention(RUNTIME)
@Target(METHOD)
public @interface AccessLimit { 
    //次数上限
    int maxCount();
    //是否需要登录
    boolean needLogin()default false;
}
```

## 添加Redis配置项

在配置文件中，加入`Redis`配置；

```properties
spring.redis.database=0
spring.redis.host=127.0.0.1
spring.redis.port=6379
spring.redis.jedis.pool.max-active=100
spring.redis.jedis.pool.max-idle=100
spring.redis.jedis.pool.min-idle=10
spring.redis.jedis.pool.max-wait=1000ms
```

注意，把`Redis`的starter在`pom`中引入。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
 </dependency>
```

## 创建拦截器

创建拦截器，所有请求都进行拦截，防刷的主要内容全部在这里。

```java
// 一堆import 这里就不贴出来了，需要的自己导入
/**
 *  处理方法上 有 AccessLimitEnum 注解的方法
 * @author java后端技术全栈
 * @date 2021/8/6 15:42
 */
@Component 
public class FangshuaInterceptor extends HandlerInterceptorAdapter {

    @Resource
    private RedisTemplate<String,Object> redisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        System.out.println("----FangshuaInterceptor-----");
        //判断请求是否属于方法的请求
        if (handler instanceof HandlerMethod) {

            HandlerMethod hm = (HandlerMethod) handler;

            //检查方法上室友有AccessLimit注解
            AccessLimit accessLimit = hm.getMethodAnnotation(AccessLimit.class);
            if (accessLimit == null) {
                return true;
            }
            //获取注解中的参数，
            int maxCount = accessLimit.maxCount();
            boolean login = accessLimit.needLogin();
            String key = request.getRequestURI();
            //防刷=同一个请求路径+同一个用户+当天
            //如果需要登录
            if (login) {
                //可以充session中获取user相关信息
                //这里的userId暂时写死，
                Long userId = 101L;
                String currentDay = format(new Date(), "yyyyMMdd");
                key += currentDay + userId;
            }else{
                //可以根据用户使用的ip+日期进行判断
            }

            //从redis中获取用户访问的次数
            Object countCache = redisTemplate.opsForValue().get(key);
            if (countCache == null) {
                //第一次访问,有效期为一天
                //时间单位自行定义
                redisTemplate.opsForValue().set(key,1,86400, TimeUnit.SECONDS);
            } else{
                Integer count = (Integer)countCache;
                if (count < maxCount) {
                    //加1
                    count++;
                    //也可以使用increment(key）方法
                    redisTemplate.opsForValue().set(key,count);
                } else {
                    //超出访问次数
                    render(response, "访问次数已达上限！");
                    return false;
                }
            }
        }
        return true;
    }
    //仅仅是为了演示哈
    private void render(HttpServletResponse response, String msg) throws Exception {
        response.setContentType("application/json;charset=UTF-8");
        OutputStream out = response.getOutputStream();
        out.write(msg.getBytes("UTF-8"));
        out.flush();
        out.close();
    }
    //日期格式
    public static String format(Date date, String formatString) {
        if (formatString == null) {
            formatString = DATE_TIME_FORMAT;
        }
        DateFormat dd = new SimpleDateFormat(formatString);
        return dd.format(date);
    }
}
```

注意：

> 判断是否为相同请求，使用：URI+userId+日期。即Redis的key=URI+userId+yyyyMMdd，缓存有效期为一天。
>
> 很多都在代码里有注释了，另外强调一下，不要吐槽代码，仅仅是演示。

## 注册拦截器

尽管上面我们已经自定义并实现好了拦截器，但还需要我们手动注册。

```java
import com.example.demo.ExceptionHander.FangshuaInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
 
    @Autowired
    private FangshuaInterceptor interceptor;
 
 
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor);
    }
}
```

这样我们的注解就正式注册到拦截器链中了，后面项目中才会有效。

## 使用注解

前面的准备都搞定了，现在来具体使用。

首先，我们创建一个简单的controller，然后，在方法上加上我们自定义的注解`AccessLimit`。

```java
import com.example.demo.result.Result;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
 
@Controller
public class FangshuaController {
    //具体请求次数由具体业务决定，以及是否需要登录
    @AccessLimit(maxCount=5, needLogin=true)
    @RequestMapping("/fangshua")
    @ResponseBody
    public Object fangshua(){
        return "请求成功";
 
    }
}
```

测试，浏览器页面上访问：`http://localhost:8080/fangshua`

前面4次返回的是：`请求成功`

超过4次后变成：`访问次数已达上限！`

>  是不是 so easy ！！！

## 总结

关于接口防刷，如果在面试中被问到，至少还是能说个123了。也建议大家手动试试，自己搞出来了更带劲儿。

好了，今天就分享到此，如果对你有帮助，记得点赞、分享、在看。

