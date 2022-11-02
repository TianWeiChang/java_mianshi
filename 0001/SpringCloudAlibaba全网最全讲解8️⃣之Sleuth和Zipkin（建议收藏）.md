SpringCloudAlibaba全网最全讲解8️⃣之Sleuth和Zipkin（建议收藏）

## 1、链路追踪简介

在大型系统的微服务化构建中，一个系统被拆分成了许多模块。这些模块负责不同的功能，组合成系统，最终可以提供丰富的功能。

在这种架构中，一次请求往往需要涉及到多个服务。互联网应用构建在不同的软件模块集上，这些软件模块，有可能是由不同的团队开发、可能使用不同的编程语言来实现、有可能布在了几千台服务器，横跨多个不同的数据中心，也就意味着这种架构形式也会存在一些问题：

1. 如何快速发现问题？
2. 如何判断故障影响范围？
3. 如何梳理服务依赖以及依赖的合理性？
4. 如何分析链路性能问题以及实时容量规划？

分布式链路追踪（Distributed Tracing），就是将一次分布式请求还原成调用链路，进行日志记录，性能监控并将一次分布式请求的调用情况集中展示。比如各个服务节点上的耗时、请求具体到达哪台机器上、每个服务节点的请求状态等等。

![image-20210507222709097](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e69af4164a444a3da694931786f73515~tplv-k3u1fbpfcp-watermark.awebp)

常见的链路追踪技术有下面这些：

1. cat：由大众点评开源，基于Java开发的实时应用监控平台，包括实时应用监控，业务监控 。 集成方案是通过代码埋点的方式来实现监控，比如： 拦截器，过滤器等。 对代码的侵入性很大，集成成本较高。风险较大。
2. zipkin：由Twitter公司开源，开放源代码分布式的跟踪系统，用于收集服务的定时数据，以解决微服务架构中的延迟问题，包括：数据的收集、存储、查找和展现。该产品结合spring-cloud-sleuth使用较为简单， 集成很方便， 但是功能较简单。
3. pinpoint：Pinpoint是韩国人开源的基于字节码注入的调用链分析，以及应用监控分析工具。特点是支持多种插件，UI功能强大，接入端无代码侵入。
4. skywalking：SkyWalking是本土开源的基于字节码注入的调用链分析，以及应用监控分析工具。特点是支持多种插件，UI功能较强，接入端无代码侵入。目前已加入Apache孵化器。
5. Sleuth：SpringCloud 提供的分布式系统中链路追踪解决方案。

Spring Cloud Alibaba 技术栈中并没有提供自己的链路追踪技术的，我们可以采用Sleuth +Zinkin来做链路追踪解决方案。

## 2、Sleuth入门

### 2.1、Sleuth简介

SpringCloud Sleuth主要功能就是在分布式系统中提供追踪解决方案。它大量借用了Google Dapper的设计， 先来了解一下Sleuth中的术语和相关概念。

- Trace

  由一组Trace Id相同的Span串联形成一个树状结构。为了实现请求跟踪，当请求到达分布式系统的入口端点时，只需要服务跟踪框架为该请求创建一个唯一的标识（即TraceId），同时在分布式系统内部流转的时候，框架始终保持传递该唯一值，直到整个请求的返回。那么我们就可以使用该唯一标识将所有的请求串联起来，形成一条完整的请求链路。

- Span

  代表了一组基本的工作单元。为了统计各处理单元的延迟，当请求到达各个服务组件的时候，也通过一个唯一标识（SpanId）来标记它的开始、具体过程和结束。通过SpanId的开始和结束时间戳，就能统计该span的调用时间，除此之外，我们还可以获取如事件的名称。请求信息等元数据。

- Annotation

  用它记录一段时间内的事件，内部使用的重要注释：

  - cs（Client Send）客户端发出请求，开始一个请求的生命。
  - sr（Server Received）服务端接受到请求开始进行处理， sr－cs = 网络延迟（服务调用的时间）。
  - ss（Server Send）服务端处理完毕准备发送到客户端，ss - sr = 服务器上的请求处理时间。
  - cr（Client Reveived）客户端接受到服务端的响应，请求结束。 cr - sr = 请求的总时间。

![image-20210507223230617](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8dc01aaa8dbd453cb20be9f69142efb5~tplv-k3u1fbpfcp-watermark.awebp)

### 2.2、集成链路追踪组件Sleuth

​    Sleuth的使用及其简单，直接引入一个依赖即可。

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

我们随便在一个服务里面打印日志，可以在控制台观察到sleuth的日志输出：

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0214a348a67a486985c2ea5ebf5e8960~tplv-k3u1fbpfcp-watermark.awebp)

日志参数详解：

[product-service,d1e92e984eaec1ff,d1e92e984eaec1ff,true]

1. 第一个值，spring.application.name的值，代表服务名
2. 第二个值，d1e92e984eaec1ff，sleuth生成的一个ID，叫Trace ID，用来标识一条请求链路，一条请求链路中包含一个Trace ID，多个Span ID
3. 第三个值，d1e92e984eaec1ff、spanID 基本的工作单元，获取元数据，如发送一个http
4. 第四个值：true，是否要将该信息输出到zipkin服务中来收集和展示。

查看日志文件并不是一个很好的方法，当微服务越来越多日志文件也会越来越多，通过Zipkin可以将日志聚合，并进行可视化展示和全文检索。

## 3、Zipkin+Sleuth整合

### 3.1、ZipKin介绍

Zipkin 是 Twitter 的一个开源项目，它基于Google Dapper实现，它致力于收集服务的定时数据，以解决微服务架构中的延迟问题，包括数据的收集、存储、查找和展现。我们可以使用它来收集各个服务器上请求链路的跟踪数据，并通过它提供的REST API接口来辅助我们查询跟踪数据以实现对分布式系统的监控程序，从而及时地发现系统中出现的延迟升高问题并找出系统性能瓶颈的根源。除了面向开发的 API 接口之外，它也提供了方便的UI组件来帮助我们直观的搜索跟踪信息和分析请求链路明细，比如：可以查询某段时间内各用户请求的处理时间等。

Zipkin 提供了可插拔数据存储方式：In-Memory、MySql、Cassandra 以及 Elasticsearch。

![image-20210508094417349](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/925bd87437ae424db8b4ca1b861fbafe~tplv-k3u1fbpfcp-watermark.awebp)

它主要由 4 个核心组件构成：

1. Collector：收集器组件，它主要用于处理从外部系统发送过来的跟踪信息，将这些信息转换为Zipkin内部处理的 Span 格式，以支持后续的存储、分析、展示等功能。
2. Storage：存储组件，它主要对处理收集器接收到的跟踪信息，默认会将这些信息存储在内存中，我们也可以修改此存储策略，通过使用其他存储组件将跟踪信息存储到数据库中。
3. RESTful API：API 组件，它主要用来提供外部访问接口。比如给客户端展示跟踪信息，或是外接系统访问以实现监控等。
4. Web UI：UI 组件， 基于API组件实现的上层应用。通过UI组件用户可以方便而有直观地查询和分析跟踪信息。

Zipkin分为两端，一个是 Zipkin服务端，一个是 Zipkin客户端，客户端也就是微服务的应用。 客户端会配置服务端的 URL 地址，一旦发生服务间的调用的时候，会被配置在微服务里面的 Sleuth 的监听器监听，并生成相应的 Trace 和 Span 信息发送给服务端。

### 3.2、 ZipKin服务端安装

> 下载ZipKin的jar包

[官网下载地址](https://link.juejin.cn?target=https%3A%2F%2Fzipkin.io%2F)，下载了以后是一个jar包。

> 通过命令启动

```
java -jar jar包名字
```

> 访问 [http://localhost:9411](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9411)

![image-20210508095432711](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc6fc7257d69478a98f46978cad7539e~tplv-k3u1fbpfcp-watermark.awebp)

### 3.3、Zipkin+Sleuth整合

> 在Shop-produuct-server和Shop-order-server中加入依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

> 在Shop-produuct-server和Shop-order-server中加入application.yml

```
spring:
  zipkin:
    base-url: http://127.0.0.1:9411/ #zipkin server的请求地址
    discoveryClientEnabled: false #让nacos把它当成一个URL，而不要当做服务名
    sleuth:
      sampler:
        probability: 1.0 #采样的百分比
```

> 访问测试

![image-20210508102949805](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e180b26d264a4e4da27f5568cffe2c5e~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210508103004885](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab557831b1ee44f3ae100992f738cb26~tplv-k3u1fbpfcp-watermark.awebp)

 DAAA

 

 

 