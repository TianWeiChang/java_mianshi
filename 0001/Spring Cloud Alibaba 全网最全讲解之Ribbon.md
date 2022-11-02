Spring Cloud Alibaba Ribbon

## 1、负载均衡简介

负载均衡就是将负载（工作任务，访问请求）进行分摊到多个操作单元（服务器,组件）上进行执行。

根据负载均衡发生位置的不同,一般分为服务端负载均衡和客户端负载均衡：

1. 服务端负载均衡指的是发生在服务提供者一方，比如常见的nginx负载均衡。
2. 客户端负载均衡指的是发生在服务请求的一方，也就是在发送请求之前已经选好了由哪个实例处理请 求。

![image-20210506110723494](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fc34f2059884136a5e5650c11b9e936~tplv-k3u1fbpfcp-watermark.awebp)

我们在微服务调用关系中一般会选择客户端负载均衡，也就是在服务调用的一方来决定服务由哪个提供 者执行。

## 2、手动实现负载均衡

### 2.1、起一个不同端口的服务

我们需要借助IDEA再启一个服务，这个服务也是ShopProductServer，但是他的端口是不同的。

在VM option中输入需要指定的端口号即可：

```
-Dserver.port=8081
```

![image-20210505192515637](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/271a5745177d46f793b55b2b208d6a20~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210505192620065](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/faae2ac58a794ae7ba132bf7ef91b219~tplv-k3u1fbpfcp-watermark.awebp)

### 2.2、修改实现类代码实现负载均衡

```
@Override
public Order getById(Long oid, Long pid) {
    // 从nacos中获取服务
    List<ServiceInstance> instances = discoveryClient.getInstances("product-service");
    // 定义一个随机数
    int index = new Random().nextInt(instances.size());
    // 随机获取一个服务
    ServiceInstance serviceInstance = instances.get(index);
    String url = serviceInstance.getHost() + ":" + serviceInstance.getPort();
    Product product = restTemplate
        .getForObject("http://"+url+"/product?productId=" + pid, Product.class);
    Order order = orderDao.getOne(oid);
    order.setUsername(order.getUsername()+",data form "+serviceInstance.getPort());
    return order;
}
```

连续测试多几次就可以发现随机的负载均衡策略实现了。

![image-20210505195610892](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f7545ff69e64d339fbb98dbdbd9314d~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210505195620979](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0588117be1384a0280a6ce4088fcbd6f~tplv-k3u1fbpfcp-watermark.awebp)

## 3、基于Ribbon实现负载均衡

### 3.1、Ribbon是什么

Ribbon是Netflix发布的开源项目，主要功能是提供客户端的软件负载均衡算法，将Netflix的中间层服务连接在一起。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。**Ribbon就是简化了上面代码的组件，其中提供了更多的负载均衡算法。**它是Spring Cloud的一个组件，它可以让我们使用一个注解就能轻松的搞定负载均衡。

### 3.2、Ribbon实现负载均衡

#### 3.2.1、添加注解

**我们需要在RestTemplate的生成方法上添加一个注解：@LoadBalance，这个注解是一个增强注解，它可以增强RestTemplate ，使他可以实现负载均衡。**

```
@Bean
@LoadBalanced
public RestTemplate getInstance(){
    return new RestTemplate();
}
```

#### 3.2.2、修改ProductService实现类的代码

为了更直观，我们直接1将application.yml中的端口号设置到实现类中。

```
@Value("${server.port}")
private String port;

@Override
public Product findById(Long productId) {
    Product product = productDao.findById(productId).get();
    product.setPname(product.getPname()+",data form"+port);
    return product;
}
```

![image-20210505201056244](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1a180f188b046eda68ee50eb88b0a6c~tplv-k3u1fbpfcp-watermark.awebp)

![image-20210505201108043](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/227d28166c274ec297134c7c9af62a13~tplv-k3u1fbpfcp-watermark.awebp)

#### 3.2.3、修改OrderService实现类的代码

```
@Override
public Order getById(Long oid, Long pid) {
    Product product = restTemplate
        .getForObject("http://product-service/product?productId=" + pid, Product.class);
    Order order = orderDao.getOne(oid);
    order.setPname(product.getPname());
    return order;
}
```

#### 3.2.4、实现原理

​    为什么使用服务的名称替代url就可以实现负载均衡了呢？

1. 通过反射获取到你传递过来的参数（假设传进来1），此时的`url`变成了：`http://product-service/product?productId=1.`
2. 按照规则进行切割，将product-service切割出来。
3. 根据从注册中心拉去过来的服务列表对应着的节点信息，假设1product-service对应着192.168.10.180:8081和`192.168.10.180:8083`。
4. 根据你1配置的负载均衡的策略去选择一个信息节点，将服务名称替换成对应的ip和端口。假设此时url变成了：

`http://192.168.10.180:8081/product?productId=1`。

使用RestTemplate去发送请求。

注意：一定要在RestTemplate的生成方法上添加@LoadBalance注解，这个注解表示将RestTemplate进行加强，只有进行了加强才会去实现上述的步骤，不然是无法实现负载均衡的。**

## 4、Ribbon支持的负载均衡策略

Ribbon内置了多种负载均衡策略,内部负载均衡的顶级接口为`com.netflix.loadbalancer.IRule`，他具体支持的负载均衡策略有：

| 策略名                    | 策略描述                                                     | 实现说明                                                     |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BestAvailableRule         | 选择一个最小的并发请求的server                               | 逐个考察Server，如果Server被tripped了，则忽略，在选择其中ActiveRequestsCount最小的server |
| AvailabilityFilteringRule | 先过滤掉故障实例，再选择并发较小的实例；                     | 使用一个AvailabilityPredicate来包含过滤server的逻辑，其实就就是检查status里记录的各个server的运行状态 |
| WeightedResponseTimeRule  | 根据相应时间分配一个weight，相应时间越长，weight越小，被选中的可能性越低。 | 一个后台线程定期的从status里面读取评价响应时间，为每个server计算一个weight。Weight的计算也比较简单responsetime 减去每个server自己平均的responsetime是server的权 |
| RetryRule                 | 对选定的负载均衡策略机上重试机制。                           | 在一个配置时间段内当选择server不成功，则一直尝试使用subRule的方式选择一个可用的server |
| RoundRobinRule            | 轮询方式轮询选择server                                       | 轮询index，选择index对应位置的server                         |
| RandomRule                | 随机选择一个server                                           | 在index上随机，选择index对应位置的server                     |
| ZoneAvoidanceRule(默认)   | 复合判断server所在区域的性能和server的可用性选择server       | 使用ZoneAvoidancePredicate和AvailabilityPredicate来判断是否选择某个server，前一个判断判定一个zone的运行性能是否可用，剔除不可用的zone（的所有server），AvailabilityPredicate用于过滤掉连接数过多的Server。 |

我们可以通过修改配置来调整Ribbon的负载均衡策略，在Shop-order-server项目的application.yml中增加如下配置：

```
product-service: # 调用的提供者的名称
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

 asdasd

 

 

 