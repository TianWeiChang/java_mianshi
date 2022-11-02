SpringCloudAlibaba全网最全讲解7️⃣之Gateway



![image-20210507115602441](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfba70cd3d234afeabb3a30d75a923e9~tplv-k3u1fbpfcp-watermark.awebp)

## 1、网关简介

大家都都知道在微服务架构中，一个系统会被拆分为很多个微服务。那么作为客户端要如何去调用这么多的微服务呢？如果没有网关的存在，我们只能在客户端记录每个微服务的地址，然后分别去调用。

![image-20210507115048632](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5fb6218a5df4f4ba41e477673e2dd5e~tplv-k3u1fbpfcp-watermark.awebp)

 这样的架构会存在许多的问题：

1. 客户端多次请求不同的微服务，增加客户端代码或配置编写的复杂性。
2. 认证复杂，每个服务都需要独立认证。
3. 存在跨域请求，在一定场景下处理相对复杂。

网关就是为了解决这些问题而生的。所谓的API网关，就是指系统的统一入口，它封装了应用程序的内部结构，为客户端提供统一服务，一些与业务本身功能无关的公共逻辑可以在这里实现，诸如认证、鉴权、监控、路由转发等等。

![image-20210507115517531](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04d70f19d89c46668279db2293008b99~tplv-k3u1fbpfcp-watermark.awebp)

## 2、常用的网关

### 2.1、Ngnix+lua

使用nginx的反向代理和负载均衡可实现对api服务器的负载均衡及高可用。

lua是一种脚本语言,可以来编写一些简单的逻辑, nginx支持lua脚本

### 2.2、Kong

基于Nginx+Lua开发，性能高，稳定，有多个可用的插件(限流、鉴权等等)可以开箱即用。

他的缺点：

1. 只支持Http协议。
2. 二次开发，自由扩展困难。
3. 提供管理API，缺乏更易用的管控、配置方式。

### 2.3、Zuul

Netflix开源的网关，功能丰富，使用JAVA开发，易于二次开发。

他的缺点：

1. 缺乏管控，无法动态配置。
2. 依赖组件较多。
3. 处理Http请求依赖的是Web容器，性能不如Nginx。

### 2.4、Spring Cloud Gateway

Spring公司为了替换Zuul而开发的网关服务，SpringCloud alibaba技术栈中并没有提供自己的网关，我们可以采用Spring Cloud Gateway来做网关

## 3、Gateway简介

Spring Cloud Gateway是Spring公司基于Spring 5.0，Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式。它的目标是替代Netflix Zuul，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控和限流。

​    他的主要功能是：

1. 进行转发重定向。
2. 在开始的时候，所有类都需要做的初始化操作。
3. 进行网络隔离。

## 4、快速入门

需求：通过浏览器访问api网关,然后通过网关将请求转发到商品微服务。

### 4.1、基础版

> 创建一个api-gateway 模块，并且导入下面的依赖。

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <parent>
    <artifactId>Shop-parent</artifactId>
    <groupId>cn.linstudy</groupId>
    <version>1.0.0</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>
  <artifactId>api-gateway</artifactId>

  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <!--gateway网关-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>
  </dependencies>

</project>
```

> 编写配置文件

```
server:
  port: 9000 # 指定网关服务的端口
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes: # 路由数组[路由 就是指定当请求满足什么条件的时候转到哪个微服务]
        - id: product_route # 当前路由的标识, 要求唯一
          uri: http://localhost:8081 # 请求要转发到的地址
          order: 1 # 路由的优先级,数字越小级别越高
          predicates: # 断言(就是路由转发要满足的条件)
            - Path=/product-serv/** # 当请求路径满足Path指定的规则时,才进行路由转发
          filters: # 过滤器,请求在传递过程中可以通过过滤器对其进行一定的修改
            - StripPrefix=1 # 转发之前去掉1层路径
```

> 测试

![image-20210507171142491](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/036c1fa7aeb54ff197a2883afe75d508~tplv-k3u1fbpfcp-watermark.awebp)

### 4.2、升级版

我们发现升级版有一个很大的问题，那就是在配置文件中写死了转发路径的地址，我们需要在注册中心来获取地址。

> 加入nacos依赖

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <parent>
    <artifactId>Shop-parent</artifactId>
    <groupId>cn.linstudy</groupId>
    <version>1.0.0</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>
  <artifactId>api-gateway</artifactId>

  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <!--gateway网关-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!--nacos客户端-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>
  </dependencies>

</project>
```

> 在主类上添加注解

```
@SpringBootApplication
@EnableDiscoveryClient
public class GateWayServerApp {
  public static void main(String[] args) {
    SpringApplication.run(GateWayServerApp.class,args);
  }
}
```

> 修改配置文件

```
server:
  port: 9000
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      discovery:
        locator:
          enabled: true # 让gateway可以发现nacos中的微服务
      routes:
        - id: product_route # 路由的名字
          uri: lb://product-service # lb指的是从nacos中按照名称获取微服务,并遵循负载均衡策略
          predicates:
            - Path=/product-serv/** # 符合这个规定的才进行1转发
          filters:
            - StripPrefix=1 # 将第一层去掉
```

我们还可以自定义多个路由规则。

```
spring:
  application:
    gateway:
      routes:
        - id: product_route
          uri: lb://product-service 
          predicates:
            - Path=/product-serv/**
          filters:
            - StripPrefix=1
        - id: order_route
          uri: lb://order-service 
          predicates:
            - Path=/order-serv/**
          filters:
            - StripPrefix=1
```

### 4.3、简写版

我们的配置文件无需写的1那么复杂就可以实现功能，有一个简写版。

```
server:
  port: 9000
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
    gateway:
      discovery:
        locator:
          enabled: true # 让gateway可以发现nacos中的微服务
```

![image-20210507201625163](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e432340351494c129d4b38aeb4a4d04f~tplv-k3u1fbpfcp-watermark.awebp)

我们发现，就发现只要按照网关地址/微服务名称/接口的格式去访问，就可以得到成功响应。

## 5、Gateway核心架构

### 5.1、基本概念

路由(Route) 是 gateway 中最基本的组件之一，表示一个具体的路由信息载体。主要定义了下面的几个信息：

1. id：路由标识符，区别于其他 Route。
2. uri：路由指向的目的地 uri，即客户端请求最终被转发到的微服务。
3. order：用于多个 Route 之间的排序，数值越小排序越靠前，匹配优先级越高。
4. predicate：断言的作用是进行条件判断，只有断言都返回真，才会真正的执行路由。
5. filter：过滤器用于修改请求和响应信息。
6. predicate：断言，用于进行条件判断，只有断言都返回真，才会真正的执行路由。

### 5.2、执行原理

![image-20201030161652819](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1890a98fb6244be99ac0c7807306ee21~tplv-k3u1fbpfcp-watermark.awebp)

1. 接收用户的请求，请求处理器交给处理器映射器，返回执行链。
2. 请求处理器去调用web处理器，在web处理器里面对我们的路径1进行处理。假设1我们的路径1是：[http://localhost:9000/product-serv/get?id=1](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9000%2Fproduct-serv%2Fget%3Fid%3D1) ，根据配置的路由规则，上本地找对应的服务信息：product-service对应的主机ip是192.168.10.130。
3. 根据1ribbon的负载均衡策略去选择一个节点，然后拼接好，将路径中的product-serv替换成192.168.10.130:8081，如果你配置了filter，那么他还会走filter。
4. 如果你没有自定义路由的话，默认Gateway会帮你把第一层去掉。网关端口从此一个`/`开始到第二个`/`开始算第一层。

![image-20210507203258953](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06c18536d2e84b9ba9e79418f1408b21~tplv-k3u1fbpfcp-watermark.awebp)

## 6、过滤器

Gateway的过滤器的作用是：是在请求的传递过程中,对请求和响应做一些手脚。

Gateway的过滤器的生命周期：

1. PRE：这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择 请求的微服务、记录调试信息等。
2. POST：这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。

​    Gateway 的Filter从作用范围可分为两种: GatewayFilter与GlobalFilter：

1. GatewayFilter：应用到单个路由或者一个分组的路由上。
2. GlobalFilter：应用到所有的路由上。

### 6.1、局部过滤器

局部过滤器是针对单个路由的过滤器。他分为内置过滤器和自定义过滤器。

#### 6.1.1、内置过滤器

在SpringCloud Gateway中内置了很多不同类型的网关路由过滤器。

##### 6.1.1.1、局部过滤器内容

| 过滤器工厂                  | 作用                                                         | 参数                                                         |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| AddRequestHeader            | 为原始请求添加Header                                         | Header的名称及值                                             |
| AddRequestParameter         | 为原始请求添加请求参数                                       | 参数名称及值                                                 |
| AddResponseHeader           | 为原始响应添加Header                                         | Header的名称及值                                             |
| DedupeResponseHeader        | 剔除响应头中重复的值                                         | 需要去重的Header名称及去重策略                               |
| Hystrix                     | 为路由引入Hystrix的断路器保护                                | HystrixCommand 的名称                                        |
| FallbackHeaders             | 为fallbackUri的请求头中添加具体的异常信息                    | Header的名称                                                 |
| PrefixPath                  | 为原始请求路径添加前缀                                       | 前缀路径                                                     |
| PreserveHostHeader          | 为请求添加一个preserveHostHeader=true的属性，路由过滤器会检查该属性以决定是否要发送原始的Host | 无                                                           |
| RequestRateLimiter          | 用于对请求限流，限流算法为令牌桶                             | keyResolver、 rateLimiter、 statusCode、 denyEmptyKey、 emptyKeyStatus |
| RedirectTo                  | 将原始请求重定向到指定的URL                                  | http状态码及重定向的url                                      |
| RemoveHopByHopHeadersFilter | 为原始请求删除IETF组织规定的一系列Header                     | 默认就会启用，可以通过配置指定仅删除哪些Header               |
| RemoveRequestHeader         | 为原始请求删除某个Header                                     | Header名称                                                   |
| RemoveResponseHeader        | 为原始响应删除某个Header                                     | Header名称                                                   |
| RewritePath                 | 重写原始的请求路径                                           | 原始路径正则表达式以及重写后路径的正则表达式                 |
| RewriteResponseHeader       | 重写原始响应中的某个Header                                   | Header名称，值的正则表达式，重写后的值                       |
| SaveSession                 | 在转发请求之前，强制执行`WebSession::save`操作               | 无                                                           |
| secureHeaders               | 为原始响应添加一系列起安全作用的响应头                       | 无，支持修改这些安全响应头的值                               |
| SetPath                     | 修改原始的请求路径                                           | 修改后的路径                                                 |
| SetResponseHeader           | 修改原始响应中某个Header的值                                 | Header名称，修改后的值                                       |
| SetStatus                   | 修改原始响应的状态码                                         | HTTP 状态码，可以是数字，也可以是字符串                      |
| StripPrefix                 | 用于截断原始请求的路径                                       | 使用数字表示要截断的路径的数量                               |
| Retry                       | 针对不同的响应进行重试                                       | retries、statuses、methods、series                           |
| RequestSize                 | 设置允许接收最大请求包的大小。如果请求包大小超过设置的值，则返回 413 Payload Too Large | 请求包大小，单位为字节，默认值为5M                           |
| ModifyRequestBody           | 在转发请求之前修改原始请求体内容                             | 修改后的请求体内容                                           |
| ModifyResponseBody          | 修改原始响应体的内容                                         | 修改后的响应体内容                                           |

##### 6.1.1.2、局部过滤器的使用

```
server:
  port: 9000
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      discovery:
        locator:
          enabled: true # 让gateway可以发现nacos中的微服务
      routes:
        - id: product_route # 路由的名字
          uri: lb://product-service # lb指的是从nacos中按照名称获取微服务,并遵循负载均衡策略
          predicates:
            - Path=/product-serv/** # 符合这个规定的才进行1转发
          filters:
            - StripPrefix=1 # 将第一层去掉
            - SetStatus=2000 # 这里使用内置的过滤器，修改返回状态
```

#### 9.6.1.2、自定义局部过滤器

很多的时候，内置过滤器没办法满足我们的需求，这个时候就必须自定义局部过滤器。我们假定一个需求是：统计订单服务调用耗时。

> 编写一个类，用于实现逻辑

名称是有固定格式xxxGatewayFilterFactory**

```
@Component
public class TimeGatewayFilterFactory extends AbstractGatewayFilterFactory<TimeGatewayFilterFactory.Config> {

  private static final String BEGIN_TIME = "beginTime";

  //构造函数
  public TimeGatewayFilterFactory() {
    super(TimeGatewayFilterFactory.Config.class);
  }

  //读取配置文件中的参数 赋值到 配置类中
  @Override
  public List<String> shortcutFieldOrder() {
    return Arrays.asList("show");
  }

  @Override
  public GatewayFilter apply(Config config) {
    return new GatewayFilter() {
      @Override
      public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        if (!config.show){
          // 如果配置类中的show为false，表示放行
          return chain.filter(exchange);
        }
        exchange.getAttributes().put(BEGIN_TIME, System.currentTimeMillis());
        /**
         *  pre的逻辑
         * chain.filter().then(Mono.fromRunable(()->{
         *     post的逻辑
         * }))
         */
        return chain.filter(exchange).then(Mono.fromRunnable(()->{
          Long startTime = exchange.getAttribute(BEGIN_TIME);
          if (startTime != null) {
            System.out.println(exchange.getRequest().getURI() + "请求耗时: " + (System.currentTimeMillis() - startTime) + "ms");
          }
        }));
      }
    };
  }

  @Setter
  @Getter
  static class Config{
    private boolean show;
  }

}
```

> 编写application.xml

```
server:
  port: 9000
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      discovery:
        locator:
          enabled: true # 让gateway可以发现nacos中的微服务
      routes:
        - id: product_route # 路由的名字
          uri: lb://product-service # lb指的是从nacos中按照名称获取微服务,并遵循负载均衡策略
          predicates:
            - Path=/product-serv/** # 符合这个规定的才进行1转发
          filters:
            - StripPrefix=1 # 将第一层去掉
        - id: order_route
          uri: lb://order-service
          predicates:
            - Path=/order-serv/**
          filters:
            - StripPrefix=1
            - Time=true
```

访问路径：[http://localhost:9000/order-serv/getById?o=1&pid=1](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9000%2Forder-serv%2FgetById%3Fo%3D1%26pid%3D1)

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d1c9daab659420a82e5f8ceddd63165~tplv-k3u1fbpfcp-watermark.awebp)

### 6.2、全局过滤器

全局过滤器作用于所有路由, 无需配置。通过全局过滤器可以实现对权限的统一校验，安全性验证等功能。SpringCloud Gateway内部也是通过一系列的内置全局过滤器对整个路由转发进行处理。

![网关全局过滤器](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0288a380387f496a83865409d4588996~tplv-k3u1fbpfcp-watermark.awebp)

  开发中的鉴权逻辑：

- 当客户端第一次请求服务时，服务端对用户进行信息认证（登录）。
- 认证通过，将用户信息进行加密形成token，返回给客户端，作为登录凭证。
- 以后每次请求，客户端都携带认证的token。
- 服务端对token进行解密，判断是否有效。

![image-20210507220009473](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09eb22f643384b7fbd31ff68dd5e9324~tplv-k3u1fbpfcp-watermark.awebp)

我们来模拟一个需求：实现统一鉴权的功能,我们需要在网关判断请求中是否包含token且，如果没有则不转发路由，有则执行正常逻辑。

> 编写全局过滤器

```
@Component
public class AuthGlobalFilter implements GlobalFilter {

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    String token = exchange.getRequest().getQueryParams().getFirst("token");
    if (StringUtils.isBlank(token)) {
      System.out.println("鉴权失败");
      exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
      return exchange.getResponse().setComplete();
    }
    return chain.filter(exchange);
  }
}
```

### 6.3、网关限流

网关是所有请求的公共入口，所以可以在网关进行限流，而且限流的方式也很多，我们本次采用前面学过的Sentinel组件来实现网关的限流。Sentinel支持对SpringCloud Gateway、Zuul等主流网关进行限流。

![image-20210507220048921](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e3dc286a9524e2cad1c856b16249dba~tplv-k3u1fbpfcp-watermark.awebp)

从1.6.0版本开始，Sentinel提供了SpringCloud Gateway的适配模块，可以提供两种资源维度的限流：

- route维度：即在Spring配置文件中配置的路由条目，资源名为对应的routeId
- 自定义API维度：用户可以利用Sentinel提供的API来自定义一些API分组

#### 9.6.3.1、网关集成Sentinel

> 添加依赖

```
<dependency>
	<groupId>com.alibaba.csp</groupId>
	<artifactId>sentinel-spring-cloud-gateway-adapter</artifactId>
</dependency>
```

> 编写配置类进行限流

配置类的本质是用代码替代nacos图形化界面限流。

```
@Configuration
public class GatewayConfiguration {
    private final List<ViewResolver> viewResolvers;
    private final ServerCodecConfigurer serverCodecConfigurer;

    public GatewayConfiguration(ObjectProvider<List<ViewResolver>> viewResolversProvider,
                                ServerCodecConfigurer serverCodecConfigurer) {
        this.viewResolvers = viewResolversProvider.getIfAvailable(Collections::emptyList);
        this.serverCodecConfigurer = serverCodecConfigurer;
    }
    // 配置限流的异常处理器
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public SentinelGatewayBlockExceptionHandler sentinelGatewayBlockExceptionHandler() {
        // Register the block exception handler for Spring Cloud Gateway.
        return new SentinelGatewayBlockExceptionHandler(viewResolvers, serverCodecConfigurer);
    }
    // 初始化一个限流的过滤器
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public GlobalFilter sentinelGatewayFilter() {
        return new SentinelGatewayFilter();
    }
    //增加对商品微服务的限流
     @PostConstruct
    private void initGatewayRules() {
        Set<GatewayFlowRule> rules = new HashSet<>();
        rules.add(new GatewayFlowRule("product_route")
                .setCount(3) // 三次
                .setIntervalSec(1) // 一秒，表示一秒钟1超过了三次就会限流
        );
        GatewayRuleManager.loadRules(rules);
    }
}
```

> 修改限流默认返回格式

如果我们不想在限流的时候返回默认的错误，那么就需要自定义错误，指定自定义的返回格式。我们只需在类中添加一段配置即可。

```
@PostConstruct
public void initBlockHandlers() {
	BlockRequestHandler blockRequestHandler = new BlockRequestHandler() {
	public Mono<ServerResponse> handleRequest(ServerWebExchange serverWebExchange, Throwable throwable) {
		Map map = new HashMap<>();
		map.put("code", 0);
		map.put("message", "接口被限流了");
			return ServerResponse.status(HttpStatus.OK).
				contentType(MediaType.APPLICATION_JSON).
				body(BodyInserters.fromValue(map));
            }
};
	GatewayCallbackManager.setBlockHandler(blockRequestHandler);
}
```

> 测试

![image-20210507221350190](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ab18d3684cd47479464f20663315077~tplv-k3u1fbpfcp-watermark.awebp)

#### 6.3.2、自定义API分组

我们可以发现，上面的这种定义，对整个服务进行了限流，粒度不够细。自定义API分组是一种更细粒度的限流规则定义，它可以实现某个方法的细粒度限流。

> 在Shop-order-server项目中添加ApiController

```
@RestController
@RequestMapping("/api")
public class ApiController {
    @RequestMapping("/hello")
    public String api1(){
        return "api";
    }
}
```

> 在GatewayConfiguration中添加配置

```
@PostConstruct
private void initCustomizedApis() {
	Set<ApiDefinition> definitions = new HashSet<>();
	ApiDefinition api1 = new ApiDefinition("order_api")
                .setPredicateItems(new HashSet<ApiPredicateItem>() {{
                    add(new ApiPathPredicateItem().setPattern("/order-serv/api/**").                 setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX));
                }});
	definitions.add(api1);
	GatewayApiDefinitionManager.loadApiDefinitions(definitions);
}
@PostConstruct
private void initGatewayRules() {
	Set<GatewayFlowRule> rules = new HashSet<>();
	rules.add(new GatewayFlowRule("product_route")
                .setCount(3)
                .setIntervalSec(1)
	);
	rules.add(new GatewayFlowRule("order_api").
                setCount(1).
                setIntervalSec(1));
    GatewayRuleManager.loadRules(rules);
}
```

> 测试

直接访问[http://localhost:8082/api/hello](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8082%2Fapi%2Fhello) 是不会发生限流的，访问[http://localhost:9000/order-serv/api/hello](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9000%2Forder-serv%2Fapi%2Fhello) 就会出现限流了。

