面试官：说说你对Hystrix服务熔断的理解

`服务熔断`类似现实生活中的“保险丝“，当某个异常条件被触发，直接熔断保险丝来起到保护电路的作用，

熔断的触发条件可以依据不同的场景有所不同，比如统计一个时间窗口内失败的调用次数。

## 断路器状态机 

![图片](https://mmbiz.qpic.cn/mmbiz_png/Wc4rFKffLwTB6GlFvj6ibxbMXbLVK6KcZsJUAk1QYtOxJb4QCYMbp0FZiciaMln256GNWbDwpmA89L7ljAqwxA1hQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`Closed`：熔断器关闭状态（所有请求返回成功）

`Open`：熔断器打开状态（调用次数累计到达阈值或者比例，熔断器打开，服务直接返回错误）

`Half Open`：熔断器半开状态（默认时间过后，进入半熔断状态，允许定量服务请求，如果调用都成功，则认为恢复了，则关闭断路器，反之打开断路器）

## 断路器原理 

不让客户端“裸调“服务器的rpc接口，而是在客户端包装一层。就在这个包装层里面，实现熔断逻辑 。

```java
public class CommandHelloWorld extends HystrixCommand<String> {

    private final String name;

    public CommandHelloWorld(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() {
        //关键点：把一个RPC调用，封装在一个HystrixCommand里面
        return "Hello " + name + "!";
    }
}
```

客户端调用：以前是直接调用远端`RPC`接口，现在是把`RPC`接口封装到`HystrixCommand`里面，它内部完成熔断逻辑：

```java
String s = new CommandHelloWorld("World").execute()
```

缺省的，上面的`HystrixCommand`是被扔到一个线程中执行的，也就是说，缺省是线程隔离策略。 还有一种策略就是不搞线程池，直接在调用者线程中执行，也就是信号量的隔离策略。 

## 熔断参数配置 

```java
circuitBreaker.requestVolumeThreshold //滑动窗口的大小，默认为20
circuitBreaker.sleepWindowInMilliseconds //过多长时间，熔断器再次检测是否开启，默认为5000，即5s钟
circuitBreaker.errorThresholdPercentage //错误率，默认50%
```

每当20个请求中，有50%失败时，熔断器就会打开，此时再调用此服务，将会直接返回失败，不再调远程服务。直到5s钟之后，重新检测该触发条件，判断是否把熔断器关闭，或者继续打开。

## 案例

```java
@RestController
@DefaultProperties(defaultFallback = "defaultFallback")
public class FeignClientTestController {

　　@HystrixCommand(commandProperties = {
        @HystrixProperty(name = "circuitBreaker.enabled",value = "true"),
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"),
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60")
　　})
　　public String serverMethod() {
　　　　return null;
　　}

　　public String defaultFallback() {
    　　return "服务器开小差了";
　　}
}
```

## 个人总结

**服务降级**：当服务内部出现异常情况，将触发降级，这个降级是每次请求都会去触发，走默认处理方法defaultFallback

**服务熔断**：在一定周期内，服务异常次数达到设定的阈值或百分比，则触发熔断，熔断后，后面的请求将都走默认处理方法`defaultFallback`。

sad 





































