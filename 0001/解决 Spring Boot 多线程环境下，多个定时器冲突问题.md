解决 Spring Boot 多线程环境下，多个定时器冲突问题

### 战术分析：

实际开发项目中一定不止一个定时器，很多场景都需要用到，而多个定时器带来的问题 : 就是如何避免多个定时器的互相冲突

### 使用场景 :

我们的订单服务，一般会有一个待支付订单，而这个待支付订单是有时间限制的，比如阿里巴巴的订单是五天，淘宝订单是一天，拼多多订单是一天，美团订单是15分钟…

基金系统中，如何同时更新多个存储分区中的基金信息…

> 总的来说，实际开发中定时器需要解决多个定时器同时并发的问题，也要解决定时器之间的冲突问题

问题不大，说到并发那就离不开多线程了…慢慢看看就懂了

### 问题场景重现 :

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudkT9hrF1Lmzxba7NDVd9cPs6ueVpQrHYFhTqicFG7KZhs96nB5pT1Tt2fNR743Y6sWYduZNojaaibw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudkT9hrF1Lmzxba7NDVd9cPrQwJmhma7XQDxYnh7neaaZoJp7fhxglxzeXt61DHOs0OBnBvWuTTRQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们清晰的看到执行结果都是scheduling-1

就此可以判定，Springboot定时器默认的是单线程的

但是问题就来了，如果在线程争夺资源后，某个线程需要比较长时间才能执行完，那其他的定时器怎么办，都只能进入等待状态，时间越久，累计等待的定时器越多，这就容易引起雪崩…

其实只需要添加一个配置类然后加注解就可以解决问题了

### 添加注解

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudkT9hrF1Lmzxba7NDVd9cPS8ic4WOOsK64xLQDH3J0ghO5SrzTPxeibql4hdHLKXShHuJ0ULppEfDw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

具体代码如下 :

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.text.SimpleDateFormat;
import java.util.Date;

@Component
public class SchedulerTaskController {
    private Logger logger= LoggerFactory.getLogger(SchedulerTaskController.class);
    private static final SimpleDateFormat dateFormat=new SimpleDateFormat("HH:mm:ss");
    private int count=0;
    @Scheduled(cron="*/6 * * * * ?")
    @Async("threadPoolTaskExecutor")
    public void process(){
        logger.info("英文:this is scheduler task runing "+(count++));
    }

    @Scheduled(fixedRate = 6000)
    @Async("threadPoolTaskExecutor")
    public void currentTime(){
        logger.info("中文:现在时间"+dateFormat.format(new Date()));
    }
}
```

配置类如下 :

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudkT9hrF1Lmzxba7NDVd9cPGZByIF1Rmab89w5s37G5PH8YIuFry5iaHvbyFGzvNfibLTt0s8Wiciawjg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

具体代码如下 :

```
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import java.util.concurrent.ThreadPoolExecutor;



/**使用多线程的时候，往往需要创建Thread类，或者实现Runnable接口，如果要使用到线程池，我们还需要来创建Executors，
 * 在使用spring中，已经给我们做了很好的支持。只要要@EnableAsync就可以使用多线程
 * 通过spring给我们提供的ThreadPoolTaskExecutor就可以使用线程池。*/
//@Configuration 表示该类是一个配置类
@Configuration
@EnableAsync
//所有的定时任务都放在一个线程池中，定时任务启动时使用不同都线程。
public class TaskScheduleConfig {
    private static final int corePoolSize = 10;         // 默认线程数
    private static final int maxPoolSize = 100;       // 最大线程数
    private static final int keepAliveTime = 10;   // 允许线程空闲时间（单位：默认为秒）,十秒后就把线程关闭
    private static final int queueCapacity = 200;   // 缓冲队列数
    private static final String threadNamePrefix = "it-is-threaddemo-"; // 线程池名前缀

    @Bean("threadPoolTaskExecutor") // bean的名称，默认为首字母小写的方法名
    public ThreadPoolTaskExecutor getDemoThread(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(keepAliveTime);
        executor.setKeepAliveSeconds(queueCapacity);
        executor.setThreadNamePrefix(threadNamePrefix);

        //线程池拒绝任务的处理策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        //初始化
        executor.initialize();

        return executor;
    }
}
```

然后我们可以很清晰地看到

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudkT9hrF1Lmzxba7NDVd9cPxXMzqiaZKmwZTicaOI9wDbBQg8ESic5eefwrH9KERecAfSwLglbvicjqtQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上，也就解决了用多线程解决Springboot多定时器冲突的问题