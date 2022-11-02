Spring Cloud Alibaba全网最全讲解6️⃣之Sentinel（建议收藏）

流量防护：Sentinel

## 1、高并发带来的问题

在微服务架构中，我们将业务拆分成一个个的服务，服务与服务之间可以相互调用，但是由于网络原因或者自身的原因，服务并不能保证服务的100%可用，如果单个服务出现问题，调用这个服务就会出现网络延迟，此时若有大量的网络涌入，会形成任务堆积，最终导致服务瘫痪。

## 2、模拟高并发

### 2.1、编写SentinelController

```java
@RestController
public class SentinelController {

  @RequestMapping("/sentinel1")
  public String sentinel1(){
    //模拟一次网络延时
    try {
      TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return "sentinel1";
  }
  @RequestMapping("/sentinel2")
  public String sentinel2(){
    return "测试高并发下的问题";
  }
}	
```

### 2.2、修改Tomcat的并发数

```yaml
  tomcat:
    threads:
      max: 10 #tomcat的最大并发值修改为10, 
```

### 2.3、使用压力测试模拟高并发

下载地址[jmeter.apache.org/](https://link.juejin.cn?target=https%3A%2F%2Fjmeter.apache.org%2F)

> 修改配置，支持中文

进入bin目录,修改jmeter.properties文件中的语言支持为language=zh_CN。

![image-20210505214428132](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/870bd1ded93c487bb928f853ab50bb5b~tplv-k3u1fbpfcp-watermark.awebp)

然后点击jmeter.bat启动软件。

![image-20210505214459838](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0be03a2ef2d49b98731f029d49d9440~tplv-k3u1fbpfcp-watermark.awebp)

> 添加线程组

![image-20201029120234401](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb6514c0ff1a44c1997a79d2d0409a3d~tplv-k3u1fbpfcp-watermark.awebp)

![image-20201029124003847](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/791200d0369e4706a11056c191932fd3~tplv-k3u1fbpfcp-watermark.awebp)

> 添加http请求

![image-20201029122851926](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9e5ab330e9d427a88ddd89af7bb74f5~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210505214830428](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e745405c80648e0a67558ec644f3c5c~tplv-k3u1fbpfcp-watermark.awebp)

> 访问

我们去访问[http://localhost:8082/sentinel2，会发现一直在转圈，这就是服务器雪崩的雏形。](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8082%2Fsentinel2%25EF%25BC%258C%25E4%25BC%259A%25E5%258F%2591%25E7%258E%25B0%25E4%25B8%2580%25E7%259B%25B4%25E5%259C%25A8%25E8%25BD%25AC%25E5%259C%2588%25EF%25BC%258C%25E8%25BF%2599%25E5%25B0%25B1%25E6%2598%25AF%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8%25E9%259B%25AA%25E5%25B4%25A9%25E7%259A%2584%25E9%259B%258F%25E5%25BD%25A2%25E3%2580%2582)

## 3、服务器雪崩

在分布式系统中,由于网络原因或自身的原因,服务一般无法保证 100% 可用。如果一个服务出现了问题，调用这个服务就会出现线程阻塞的情况，此时若有大量的请求涌入，就会出现多条线程阻塞等待，进而导致服务瘫痪。

由于服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的 “**雪崩效应**” 。

​    服务器一步步雪崩的流程如下：

![image-20210506110941736](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0615225582243148bacb0a9074fdf05~tplv-k3u1fbpfcp-watermark.awebp)

服务器的雪崩效应其实就是由于某个微小的服务挂了,导致整一大片的服务都不可用.类似生活中的雪崩效应,由于落下的最后一片雪花引发了雪崩的情况.

雪崩发生的原因多种多样，有不合理的容量设计，或者是高并发下某一个方法响应变慢，亦或是某台机器的资源耗尽。我们无法完全杜绝雪崩源头的发生，只有做好足够的容错，保证在一个服务发生问题，不会影响到其它服务的正常运行。

雪崩发生的原因多种多样，有不合理的容量设计，或者是高并发下某一个方法响应变慢，亦或是某台机器的资源耗尽。我们无法完全杜绝雪崩源头的发生，只有做好足够的容错，保证在一个服务发生问题，不会影响到其它服务的正常运行。也就是＂雪落而不雪崩＂。

## 4、常见解决方案

要防止雪崩的扩散，我们就要做好服务的容错，容错说白了就是保护自己不被猪队友拖垮的一些措施, 下面介绍常见的服务容错思路和组件。

> 常见的容错思路有隔离、超时、限流、熔断、降级这几种。

### 4.1、隔离机制

比如服务A内总共有100个线程, 现在服务A可能会调用服务B,服务C,服务D.我们在服务A进行远程调用的时候,给不同的服务分配固定的线程,不会把所有线程都分配给某个微服务. 比如调用服务B分配30个线程,调用服务C分配30个线程，调用服务D分配40个线程. 这样进行资源的隔离，保证即使下游某个服务挂了，也不至于把服务A的线程消耗完。比如服务B挂了，这时候最多只会占用服务A的30个线程,服务A还有70个线程可以调用服务C和服务D。

![image-20201029142100450](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ae0971d6f8642ed8e94bfd1eb93ebbf~tplv-k3u1fbpfcp-watermark.awebp)

### 4.2、超时机制

在上游服务调用下游服务的时候，设置一个最大响应时间，如果超过这个时间，下游未作出反应，就断开请求，释放掉线程。

![image-20201029143237828](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a44318636c3540a6bbde2058a672fb0f~tplv-k3u1fbpfcp-watermark.awebp)

### 4.3、限流机制

限流就是限制系统的输入和输出流量已达到保护系统的目的。为了保证系统的稳固运行,一旦达到的需要限制的阈值,就需要限制流量并采取少量措施以完成限制流量的目的。

![image-20201029143206491](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b38314149e64fc69de72457ad62f104~tplv-k3u1fbpfcp-watermark.awebp)

### 4.4、熔断机制

在互联网系统中，当下游服务因访问压力过大而响应变慢或失败，上游服务为了保护系统整体的可用性，可以暂时切断对下游服务的调用。这种牺牲局部，保全整体的措施就叫做熔断。

服务熔断一般有三种状态：

1. 熔断关闭状态（Closed）：服务没有故障时，熔断器所处的状态，对调用方的调用不做任何限制。
2. 熔断开启状态（Open）：后续对该服务接口的调用不再经过网络，直接执行本地的fallback方法。
3. 半熔断状态（Half-Open）：尝试恢复服务调用，允许有限的流量调用该服务，并监控调用成功率。如果成功率达到预期，则说明服务已恢复，进入熔断关闭状态；如果成功率仍旧很低，则重新进入熔断关闭状态。

![image-20201029143128555](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18c5de2a8bb040eabb9a67ec65067f84~tplv-k3u1fbpfcp-watermark.awebp)

### 4.5、降级机制

降级其实就是为服务提供一个兜底方案，一旦服务无法正常调用，就使用兜底方案。

![image-20201029143456888](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee34256b0d814b95929cb89f1811ea06~tplv-k3u1fbpfcp-watermark.awebp)

## 5、常见的熔断组件

### 5.1、Hystrix

Hystrix是由Netflflix开源的一个延迟和容错库，用于隔离访问远程系统、服务或者第三方库，防止级联失败，从而提升系统的可用性与容错性。

### 5.2、Resilience4J

Resilicence4J一款非常轻量、简单，并且文档非常清晰、丰富的熔断工具，这也是Hystrix官方推荐的替代产品。不仅如此，Resilicence4j还原生支持Spring Boot 1.x/2.x，而且监控也支持和prometheus等多款主流产品进行整合。

### 5.3、Sentinel

Sentinel 是阿里巴巴开源的一款断路器实现，本身在阿里内部已经被大规模采用，非常稳定。

## 6、Sentinel实战

### 6.1、什么是Sentinel

Sentinel (分布式系统的流量防卫兵) 是阿里开源的一套用于**服务容错**的综合性解决方案。它以流量为切入点, 从**流量控、熔断降级、系统负载保护**等多个维度来保护服务的稳定性。

Sentinel 具有以下特征：

1. **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景, 例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
2. **完备的实时监控**：Sentinel 提供了实时的监控功能。通过控制台可以看到接入应用的单台机器秒级数据, 甚至 500 台以下规模的集群的汇总运行情况。
3. **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架/库的整合模块, 例如与 SpringCloud、Dubbo、gRPC 的整合。只需要引入相应的依赖并进行简单的配置即可快速地接入Sentinel。

### 6.2、Sentinel组成部分

Sentinel分为两部分：

1. 核心库（Java 客户端）不依赖任何框架/库,能够运行于所有 Java 运行时环境，同时对 Dubbo /Spring Cloud 等框架也有较好的支持。
2. 控制台（Dashboard）基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等应用容器。

## 7、集成Sentinel

微服务集成Sentinel非常简单, 只需要加入Sentinel的依赖即可。

### 7.1、加入依赖

```xml
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

### 7.2、编写控制器

```java
@RestController
public class SentinelController {

  @RequestMapping("/sentinel1")
  public String sentinel1(){
    //模拟一次网络延时
    try {
      TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return "sentinel1";
  }
  @RequestMapping("/sentinel2")
  public String sentinel2(){
    return "测试高并发下的问题";
  }
}
```

### 7.3、安装Sentinel控制台

> 下载jar包

Sentinel 提供一个轻量级的控制台, 它提供机器发现、单机资源实时监控以及规则管理等功能。我们需要去官网下载。

> 修改application.yml

```yaml
spring:
  cloud:
    sentinel: 
      transport: 
        port: 9999 #跟控制台交流的端口,随意指定一个未使用的端口即可 
        dashboard: localhost:8080 # 指定控制台服务的地址
```

> 启动控制台

```
# 直接使用jar命令启动项目(控制台本身是一个SpringBoot项目) 
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.0.jar
```

> 测试

通过浏览器访问localhost:8080 进入控制台 ( 默认用户名密码是 sentinel/sentinel )

![image-20210506114024909](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99edb48e293e48b8bff5d2a4f19321c7~tplv-k3u1fbpfcp-watermark.awebp)

### 7.4、控制台的原理

Sentinel的控制台其实就是一个Spring Boot编写的程序。我们需要将我们的微服务程序注册到控制台上,即在微服务中指定控制台的地址, 并且还要开启一个跟控制台传递数据的端口, 控制台也可以通过此端口调用微服务中的监控程序获取微服务的各种信息。

![image-20210506113704987](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99fd32a04ca14b16a157da65099ad390~tplv-k3u1fbpfcp-watermark.awebp)

## 8、实现一个接口限流

> 点击簇点链路->流控

![image-20210506114220936](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b27da15090e24ddd91bd78ed339c6ca6~tplv-k3u1fbpfcp-watermark.awebp)

> 在单机阈值中写数值

在单机阈值填写一个数值，表示每秒上限的请求数

![image-20210506114332147](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/649df97501ee450caa835a527cf7ac42~tplv-k3u1fbpfcp-watermark.awebp)

> 测试

快速访问几次，可以发现出错了。

![image-20210506114408464](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/371fc8c0e8584f6cae80b0259bf7c9dd~tplv-k3u1fbpfcp-watermark.awebp)

## 9、Sentinel基本概念和功能

### 9.1、基本概念

#### 9.1.1、资源

资源就是Sentinel要保护的东西。资源是 Sentinel 的关键概念。它可以是 Java 应用程序中的任何内容，可以是一个服务，也可以是一个方法，甚至可以是一段代码。

我们上面例子的一个sentinel2方法就是一个资源。

#### 9.1.2、规则

规则就是用来定义如何进行保护资源的。作用在资源之上, 定义以什么样的方式保护资源，主要包括流量控制规则、熔断降级规则以及系统保护规则。

我们上面的例子给sentinel2增加流控规则，限制了sentinel2的流量。

### 9.2、重要功能

![image-20210506115159276](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eea13c11d25047cabe4cee0deacc7349~tplv-k3u1fbpfcp-watermark.awebp)

Sentinel的主要功能就是容错，主要体现为下面这三个：

- 流量控制

  流量控制在网络传输中是一个常用的概念，它用于调整网络包的数据。任意时间到来的请求往往是 随机不可控的，而系统的处理能力是有限的。我们需要根据系统的处理能力对流量进行控制。 Sentinel 作为一个调配器，可以根据需要把随机的请求调整成合适的形状。

- 熔断降级

  当检测到调用链路中某个资源出现不稳定的表现，例如请求响应时间长或异常比例升高的时候，则 对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联故障。

  Sentinel 对这个问题采取了两种手段：

  - 通过并发线程数进行限制：Sentinel 通过限制资源并发线程的数量，来减少不稳定资源对其它资源的影响。当某个资源出现不稳定的情况下，例如响应时间变长，对资源的直接影响就是会造成线程数的逐步堆积。当线程数在特定资源上堆积到一定的数量之后，对该资源的新请求就会被拒绝。堆积的线程完成任务后才开始继续接收求。
  - 通过响应时间对资源进行降级：除了对并发线程数进行控制以外，Sentinel 还可以通过响应时间来快速降级不稳定的资源。当依赖的资源出现响应时间过长后，所有对该资源的访问都会被直接拒绝，直到过了指定的时间窗口之后才重新恢复。

- 系统负载保护

  Sentinel 同时提供系统维度的自适应保护能力。当系统负载较高的时候，如果还持续让 请求进入可能会导致系统崩溃，无法响应。在集群环境下，会把本应这台机器承载的流量转发到其 它的机器上去。如果这个时候其它的机器也处在一个边缘状态的时候，Sentinel 提供了对应的保 护机制，让系统的入口流量和系统的负载达到一个平衡，保证系统在能力范围之内处理最多的请 求。

​    **总结：我们需要做的事情，就是在Sentinel的资源上配置各种各样的规则，来实现各种容错的功能。**

## 10、Sentinel流控规则

流量控制，其原理是监控应用流量的`QPS`(每秒查询率) 或并发线程数等指标，当达到指定的阈值时对流量进行控制，以避免被瞬时的流量高峰冲垮，从而保障应用的高可用性。

![image-20210506140317192](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa01597f55a240a8a08bdf8dccfd53f9~tplv-k3u1fbpfcp-watermark.awebp)

​    资源名：唯一名称，默认是请求路径，可自定义。

​    针对来源：指定对哪个微服务进行限流，默认指default，意思是不区分来源，全部限制。

​    阈值类型/单机阈值：

- QPS（每秒请求数量）: 当调用该接口的QPS达到阈值的时候，进行限流。
- 线程数：当调用该接口的线程数达到阈值的时候，进行限流。

### 10.1、线程数限流

前面我们已经测试过了QPS限流，所以现在我们改为线程数限流。

#### 10.1.1、添加流控规则

![image-20210506141529265](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a498668ec5a4a80be6aed5ad2870271~tplv-k3u1fbpfcp-watermark.awebp)

#### 10.1.2、在Jmeter中新增线程

![image-20210506141801025](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ad1dc4742e6466d908d9b50ccc965e3~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210506141733911](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7057cd191f4f453691cd4e755661591c~tplv-k3u1fbpfcp-watermark.awebp)

#### 10.1.3、测试

![image-20210506141941996](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af4fa905849b4dc0aecb60051960ed74~tplv-k3u1fbpfcp-watermark.awebp)

### 10.2、流控模式

点击上面设置流控规则的**编辑**按钮，然后在编辑页面点击**高级选项**，会看到有流控模式一栏。

![image-20210506142200222](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e256a308c75b4890ac3917c58c03dcb7~tplv-k3u1fbpfcp-watermark.awebp)

他有三种流控模式：

1. 直接（默认）：接口达到限流条件时，开启限流。
2. 关联：当关联的资源达到限流条件时，开启限流 [适合做应用让步]。
3. 链路：当从某个接口过来的资源达到限流条件时，开启限流

#### 10.2.1、关联流控模式

关联流控模式指的是，当指定接口关联的接口达到限流条件时，开启对指定接口开启限流。

比如：当两个资源之间具有资源争抢或者依赖关系的时候，这两个资源便具有了关联。比如对数据库同一个字段的读操作和写操作存在争抢，读的速度过高会影响写得速度，写的速度过高会影响读的速度。如果放任读写操作争抢资源，则争抢本身带来的开销会降低整体的吞吐量。可使用关联限流来避免具有关联关系的资源之间过度的争抢。

我们测试的时候可以关联sentinel1这个资源。

![image-20210506142740700](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5d1c89a43a34a67af811be87e19ca52~tplv-k3u1fbpfcp-watermark.awebp)

我们使用Jmeter软件连续向/sentinel1连续发送请求，注意QPS一定要大于2，我们访问/sentinel2的时候发现被限流了。

![image-20210506143303102](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59a39dda64654543bdc272617106d4cf~tplv-k3u1fbpfcp-watermark.awebp)

#### 10.2.2、链路流控模式

链路流控模式指的是，当从某个接口过来的资源达到限流条件时，开启限流。它的功能有点类似于针对来源配置项，区别在于：针对来源是针对上级微服务，而链路流控是针对上级接口，也就是说它的粒度更细。

> 修改application.yml

```
spring:
  cloud:
    sentinel:
      web-context-unify: false
```

> TraceServiceImpl

```
@Service
@Slf4j
public class TraceServiceImpl {
    @SentinelResource(value = "tranceService")
    public void tranceService(){
        log.info("调用tranceService方法");
    }
}
```

> 新增TraceController

```
@RestController
public class TraceController {
    @Autowired
    private TraceServiceImpl traceService;
    @RequestMapping("/trace1")
    public String trace1(){
        traceService.tranceService();
        return "trace1";
    }
    @RequestMapping("/trace2")
    public String trace2(){
        traceService.tranceService();
        return "trace2";
    }
}
```

> 重新启动订单服务并添加链路流控规则

![image-20210506144520611](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af1f650aa1764dada19b99c32b36d845~tplv-k3u1fbpfcp-watermark.awebp)

> 测试

我们去访问 /trace1 和 /trace2 访问, 发现/trace2没问题, /trace1的被限流了。

![image-20210506144645394](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/104f6e88a0f04cbabd3b932d42b13423~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210506144629866](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32f96fb8a68a40e4a6375c6694fd3327~tplv-k3u1fbpfcp-watermark.awebp)

### 10.3、流控效果

1. 快速失败（默认）: 直接失败，抛出异常，不做任何额外的处理，是最简单的效果。
2. Warm Up：它从开始阈值到最大QPS阈值会有一个缓冲阶段，一开始的阈值是最大QPS阈值的 1/3，然后慢慢增长，直到最大阈值，适用于将突然增大的流量转换为缓步增长的场景。
3. 排队等待：让请求以均匀的速度通过，单机阈值为每秒通过数量，其余的排队等待； 它还会让设 置一个超时时间，当请求超过超时间时间还未处理，则会被丢弃。

## 11、Sentinel降级规则

​    降级规则就是设置当满足什么条件的时候，对服务进行降级。Sentinel提供了三个衡量条件：

- **慢调用比例**: 选择以慢调用比例作为阈值，需要设置允许的慢调用 RT（即最大的响应时间），请求的响应时间大于该值则统计为慢调用。当单位统计时长内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。
- **异常比例**: 当单位统计时长内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。异常比率的阈值范围是 `[0.0, 1.0]`，代表 0% - 100%。
- **异常数**：当单位统计时长内的异常数目超过阈值之后会自动进行熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。

### 11.1、慢调用比例

> 新增FallBackController

```
@RestController
@Slf4j
public class FallBackController {
    @RequestMapping("/fallBack1")
    public String fallBack1(){
        try {
            log.info("fallBack1执行业务逻辑");
            //模拟业务耗时
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "fallBack1";
    }
}
```

> 新增降级规则

![image-20210506150006903](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2856c1a958704b1fb1564040f9dfeb0f~tplv-k3u1fbpfcp-watermark.awebp)

上面配置表示，如果在1S之内,有【超过1个的请求】且这些请求中【响应时间>最大RT】的【请求数量比例>10%】，就会触发熔断，在接下来的10s之内都不会调用真实方法，直接走降级方法。

​    比如: 最大RT=900,比例阈值=0.1,熔断时长=10,最小请求数=10

- 情况1: 1秒内的有20个请求，只有10个请求响应时间>900ms, 那慢调用比例=0.5，这种情况就会触发熔断。
- 情况2: 1秒内的有20个请求，只有1个请求响应时间>900ms, 那慢调用比例=0.05，这种情况不会触发熔断。
- 情况3: 1秒内的有8个请求，只有6个请求响应时间>900ms, 那慢调用比例=0.75，这种情况不会触发熔断，因为最小请求数这个条件没有满足。

**我们做实验的时候把最小请求数设置为1，因为在1秒内，手动操作很难在1s内发两个请求过去，所以要做出效果,最好把最小请求数设置为1。**

### 11.2、异常数

> 在方法中新增一个异常

​    在Shop-order-server项目的FallBackController.java类新增fallBack3方法。

```
  @RequestMapping("/fallBack3")
  public String fallBack3(String name){
    log.info("fallBack3执行业务逻辑");
    if("xiaolin".equals(name)){
      throw new RuntimeException();
    }
    return "fallBack3";
  }
```

> 配置降级规则

![image-20210506150810028](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53d14a054ce2490f851fab953c483e1b~tplv-k3u1fbpfcp-watermark.awebp)

在1s之内，,有【超过3个的请求】，请求中超过2个请求出现异常就会触发熔断，熔断时长为10s。

> 测试

![image-20210506150845623](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee38153954ed462aa992c7d92fbd6f18~tplv-k3u1fbpfcp-watermark.awebp)

## 12、Sentinel热点规则

热点即经常访问的数据。很多时候我们希望统计某个热点数据中访问频次最高的 Top K 数据，并对其访问进行限制。比如：

- 商品 ID 为参数，统计一段时间内最常购买的商品 ID 并进行限制。
- 用户 ID 为参数，针对一段时间内频繁访问的用户 ID 进行限制.

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。

> 新增HotSpotController

**一定需要在请求方法上贴@SentinelResource注解，否则热点规则无效**

```
@RestController
@Slf4j
public class HotSpotController {
    @RequestMapping("/hotSpot1")
    @SentinelResource(value = "hotSpot1")
    public String hotSpot1(Long productId){
        log.info("访问编号为:{}的商品",productId);
        return "hotSpot1";
    }
}
```

> 新增热点规则

​    因为我们就一个参数，所以参数索引是0。

![image-20210506152252494](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5955a2bd982e496c8d96cea3634d48f8~tplv-k3u1fbpfcp-watermark.awebp)

> 访问一下/hotSpot1，再编辑热点规则

**添加后再去热点规则中编辑规则，在编辑之前一定要先访问一下/hotSpot1，不然参数规则无法新增。**

![image-20210506152452142](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94f51f3938044afa9f518fda1dd3fa19~tplv-k3u1fbpfcp-watermark.awebp)

> 新增参数规则

![image-20210506153033157](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ead98bb1d41d42de873c75c109b84937~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210506153054676](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fab8fe37d3694e00bda7909ce1c0fb0c~tplv-k3u1fbpfcp-watermark.awebp)

> 测试

访问：[http://localhost:8082/hotSpot1?productId=2，无论怎么样访问都无济于事。](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8082%2FhotSpot1%3FproductId%3D2%25EF%25BC%258C%25E6%2597%25A0%25E8%25AE%25BA%25E6%2580%258E%25E4%25B9%2588%25E6%25A0%25B7%25E8%25AE%25BF%25E9%2597%25AE%25E9%2583%25BD%25E6%2597%25A0%25E6%25B5%258E%25E4%25BA%258E%25E4%25BA%258B%25E3%2580%2582)

访问：[http://localhost:8082/hotSpot1?productId=1，多次访问后会降级。](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8082%2FhotSpot1%3FproductId%3D1%25EF%25BC%258C%25E5%25A4%259A%25E6%25AC%25A1%25E8%25AE%25BF%25E9%2597%25AE%25E5%2590%258E%25E4%25BC%259A%25E9%2599%258D%25E7%25BA%25A7%25E3%2580%2582)

![image-20210506153210123](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfe42a82983148c8bfbbb016e32e786e~tplv-k3u1fbpfcp-watermark.awebp)

## 13、Sentinel授权规则

很多时候，我们需要根据调用来源来判断该次请求是否允许放行，这时候可以使用 Sentinel 的来源访问控制的功能。来源访问控制根据资源的请求来源（origin）限制资源是否通过：

1. 若配置白名单，则只有请求来源位于白名单内时才可通过；
2. 若配置黑名单，则请求来源位于黑名单时不通过，其余的请求通过。

> 新增一个工具类，定义请求来源如何获取

```
@Component
public class RequestOriginParserDefinition implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest request) {
        /**
         *  定义从请求的什么地方获取来源信息
         *  比如我们可以要求所有的客户端需要在请求头中携带来源信息
         */
        String serviceName = request.getParameter("serviceName");
        return serviceName;
    }
}
```

> 新增AuthController

```
@RestController
@Slf4j
public class AuthController {
    @RequestMapping("/auth1")
  public String auth1(String serviceName){
    log.info("应用:{},访问接口",serviceName);
    return "auth1";
  }
}
```

> 新增规则

![image-20210506155905910](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1def0eb7b91e45919e3bf3f04b71f6ab~tplv-k3u1fbpfcp-watermark.awebp)

> 测试

访问[http://localhost:8082/auth1?serviceName=pc不能访问](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8082%2Fauth1%3FserviceName%3Dpc%25E4%25B8%258D%25E8%2583%25BD%25E8%25AE%25BF%25E9%2597%25AE)

访问[http://localhost:8082/auth1?serviceName=app可以访问](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8082%2Fauth1%3FserviceName%3Dapp%25E5%258F%25AF%25E4%25BB%25A5%25E8%25AE%25BF%25E9%2597%25AE)

![image-20210506160038198](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa168b50214e491fb4bb0ee2d4948066~tplv-k3u1fbpfcp-watermark.awebp)

## 14、系统规则

系统保护规则是从应用级别的入口流量进行控制，从单台机器的总体 Load、RT、入口 QPS 、CPU使用率和线程数五个维度监控应用数据，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

系统保护规则是应用整体维度的，而不是资源维度的，并且仅对入口流量 (进入应用的流量) 生效。

- Load（仅对 Linux/Unix-like 机器生效）：当系统 load1 超过阈值，且系统当前的并发线程数超过系统容量时才会触发。
- 系统保护。系统容量由系统的 maxQps * minRt 计算得出。设定参考值一般是 CPU cores * 2.5。
- RT：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
- 线程数：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
- 入口 QPS：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。
- CPU使用率：当单台机器上所有入口流量的 CPU使用率达到阈值即触发系统保护。

## 15、自定义异常返回

常见的异常大致分为这几类:

1. FlowException：限流异常 。
2. DegradeException：降级异常。
3. ParamFlowException：参数限流异常。
4. AuthorityException：授权异常。
5. SystemBlockException：系统负载异常。

在Shop-order-server项目中定义异常返回处理类。

```
@Component
public class ExceptionHandlerPage implements BlockExceptionHandler {
  @Override
  public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {
    response.setContentType("application/json;charset=utf-8");
    ResultData data = null;
    if (e instanceof FlowException) {
      data = new ResultData(-1, "接口被限流了");
    } else if (e instanceof DegradeException) {
      data = new ResultData(-2, "接口被降级了");
    }else if (e instanceof ParamFlowException) {
      data = new ResultData(-3, "参数限流异常");
    }else if (e instanceof AuthorityException) {
      data = new ResultData(-4, "授权异常");
    }else if (e instanceof SystemBlockException) {
      data = new ResultData(-5, "接口被降级了...");
    }
    response.getWriter().write(JSON.toJSONString(data));
  }
}
@Data
@AllArgsConstructor//全参构造
@NoArgsConstructor
//无参构造
class ResultData {
  private int code;
  private String message;
}
```

![image-20210506171904602](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/444b1a3135714e04973fea9da48a9bfe~tplv-k3u1fbpfcp-watermark.awebp)

## 16、`@SentinelResource`的使用

`@SentinelResource `用于定义资源，并提供可选的异常处理和 fallback 配置项。主要参数如下：

| 属性                             | 作用                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| value                            | 资源名称，必需项（不能为空）                                 |
| entryType                        | entry 类型，可选项（默认为 `EntryType.OUT`）                 |
| blockHandler`/`blockHandlerClass | `blockHandler` 对应处理 `BlockException` 的函数名称，可选项。blockHandler 函数访问范围需要是 `public`，返回类型需要与原方法相匹配，参数类型需要和原方法相匹配并且最后加一个额外的参数，类型为 `BlockException`。blockHandler 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `blockHandlerClass` 为对应的类的 `Class` 对象，注意对应的函数必需为 static 函数，否则无法解析。 |
| fallback`/`fallbackClass         | fallback 函数名称，可选项，用于在抛出异常的时候提供 fallback 处理逻辑。fallback 函数可以针对所有类型的异常（除了 `exceptionsToIgnore` 里面排除掉的异常类型）进行处理。fallback 函数签名和位置要求： 1. 返回值类型必须与原函数返回值类型一致；  2.方法参数列表需要和原函数一致，或者可以额外多一个 `Throwable` 类型的参数用于接收对应的异常。 3.fallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `fallbackClass` 为对应的类的 `Class` 对象，注意对应的函数必需为 static 函数，否则无法解析。 |
| `defaultFallback`                | 默认的 fallback 函数名称，可选项，通常用于通用的 fallback 逻辑（即可以用于很多服务或方法）。默认 fallback 函数可以针对所有类型的异常（除了 `exceptionsToIgnore` 里面排除掉的异常类型）进行处理。若同时配置了 fallback 和 defaultFallback，则只有 fallback 会生效。defaultFallback 函数签名要求： 1. 返回值类型必须与原函数返回值类型一致； 2. 方法参数列表需要为空，或者可以额外多一个 `Throwable` 类型的参数用于接收对应的异常。  3. defaultFallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `fallbackClass` 为对应的类的 `Class` 对象，注意对应的函数必需为 static 函数，否则无法解析。 |
| `exceptionsToIgnore`             | 用于指定哪些异常被排除掉，不会计入异常统计中，也不会进入 fallback 逻辑中，而是会原样抛出。 |

直接将限流或降级后执行的方法。

![image-20210506173330519](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02a995bcfe05462792ba293ff14c62ca~tplv-k3u1fbpfcp-watermark.awebp)

## 17、 Sentinel规则持久化

通过前面的讲解，我们已经知道，可以通过Dashboard来为每个Sentinel客户端设置各种各样的规则，但是这里有一个问题，就是这些规则默认是存放在内存中，极不稳定，所以需要将其持久化。

本地文件数据源会定时轮询文件的变更，读取规则。这样我们既可以在应用本地直接修改文件来更新规则，也可以通过 Sentinel 控制台推送规则。以本地文件数据源为例，推送过程如下图所示：

![image-20201030135911029](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c5a2bfd8bec487b85cdb33ae4847ca5~tplv-k3u1fbpfcp-watermark.awebp)

首先 Sentinel 控制台通过 API 将规则推送至客户端并更新到内存中，接着注册的写数据源会将新的规则保存到本地的文件中。

> 编写处理类

```java
public class FilePersistence implements InitFunc {
    @Value("${spring.application.name}")
    private String appcationName;

    @Override
    public void init() throws Exception {
        String ruleDir = System.getProperty("user.home") + "/sentinel-rules/" + appcationName;
        String flowRulePath = ruleDir + "/flow-rule.json";
        String degradeRulePath = ruleDir + "/degrade-rule.json";
        String systemRulePath = ruleDir + "/system-rule.json";
        String authorityRulePath = ruleDir + "/authority-rule.json";
        String paramFlowRulePath = ruleDir + "/param-flow-rule.json";

        this.mkdirIfNotExits(ruleDir);
        this.createFileIfNotExits(flowRulePath);
        this.createFileIfNotExits(degradeRulePath);
        this.createFileIfNotExits(systemRulePath);
        this.createFileIfNotExits(authorityRulePath);
        this.createFileIfNotExits(paramFlowRulePath);

        // 流控规则
        ReadableDataSource<String, List<FlowRule>> flowRuleRDS = new FileRefreshableDataSource<>(
                flowRulePath,
                flowRuleListParser
        );
        FlowRuleManager.register2Property(flowRuleRDS.getProperty());
        WritableDataSource<List<FlowRule>> flowRuleWDS = new FileWritableDataSource<>(
                flowRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerFlowDataSource(flowRuleWDS);

        // 降级规则
        ReadableDataSource<String, List<DegradeRule>> degradeRuleRDS = new FileRefreshableDataSource<>(
                degradeRulePath,
                degradeRuleListParser
        );
        DegradeRuleManager.register2Property(degradeRuleRDS.getProperty());
        WritableDataSource<List<DegradeRule>> degradeRuleWDS = new FileWritableDataSource<>(
                degradeRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerDegradeDataSource(degradeRuleWDS);

        // 系统规则
        ReadableDataSource<String, List<SystemRule>> systemRuleRDS = new FileRefreshableDataSource<>(
                systemRulePath,
                systemRuleListParser
        );
        SystemRuleManager.register2Property(systemRuleRDS.getProperty());
        WritableDataSource<List<SystemRule>> systemRuleWDS = new FileWritableDataSource<>(
                systemRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerSystemDataSource(systemRuleWDS);

        // 授权规则
        ReadableDataSource<String, List<AuthorityRule>> authorityRuleRDS = new FileRefreshableDataSource<>(
                authorityRulePath,
                authorityRuleListParser
        );
        AuthorityRuleManager.register2Property(authorityRuleRDS.getProperty());
        WritableDataSource<List<AuthorityRule>> authorityRuleWDS = new FileWritableDataSource<>(
                authorityRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerAuthorityDataSource(authorityRuleWDS);

        // 热点参数规则
        ReadableDataSource<String, List<ParamFlowRule>> paramFlowRuleRDS = new FileRefreshableDataSource<>(
                paramFlowRulePath,
                paramFlowRuleListParser
        );
        ParamFlowRuleManager.register2Property(paramFlowRuleRDS.getProperty());
        WritableDataSource<List<ParamFlowRule>> paramFlowRuleWDS = new FileWritableDataSource<>(
                paramFlowRulePath,
                this::encodeJson
        );
        ModifyParamFlowRulesCommandHandler.setWritableDataSource(paramFlowRuleWDS);
    }

    private Converter<String, List<FlowRule>> flowRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<FlowRule>>() {
            }
    );
    private Converter<String, List<DegradeRule>> degradeRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<DegradeRule>>() {
            }
    );
    private Converter<String, List<SystemRule>> systemRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<SystemRule>>() {
            }
    );

    private Converter<String, List<AuthorityRule>> authorityRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<AuthorityRule>>() {
            }
    );

    private Converter<String, List<ParamFlowRule>> paramFlowRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<ParamFlowRule>>() {
            }
    );

    private void mkdirIfNotExits(String filePath) throws IOException {
        File file = new File(filePath);
        if (!file.exists()) {
            file.mkdirs();
        }
    }

    private void createFileIfNotExits(String filePath) throws IOException {
        File file = new File(filePath);
        if (!file.exists()) {
            file.createNewFile();
        }
    }
    private <T> String encodeJson(T t) {
        return JSON.toJSONString(t);
    }
}
```

之后我们重启发现配置的规则还在，说明持久化成功！

 

 

 

 