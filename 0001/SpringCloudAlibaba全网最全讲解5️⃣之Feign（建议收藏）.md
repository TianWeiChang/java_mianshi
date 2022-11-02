SpringCloudAlibaba全网最全讲解5️⃣之Feign（建议收藏）

## Feign简介

Feign是Spring Cloud提供的一个声明式的伪Http客户端， 它使得调用远程服务就像调用本地服务一样简单， 只需要创建一个接口并添加一个注解即可。

Nacos很好的兼容了Feign， Feign默认集成了 Ribbon， 所以在Nacos下使用Fegin默认就实现了负载均衡的效果。

## Feign实战

### 添加依赖

在shop-order-server项目的pom文件加入Fegin的依赖。

```
<!--fegin组件-->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### 添加注解

我们需要在启动类上添加`@EnableFeignClients`注解，只有有了这个注解才会扫描。

```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients // 支持Feign
public class ShopOrderServerApp {
  public static void main(String[] args) {
    SpringApplication.run(ShopOrderServerApp.class,args);
  }

  @Bean
  @LoadBalanced
  public RestTemplate getInstance(){
    return new RestTemplate();
  }
}
```

### 新增`ProductFeignApi`

​    在`Shop-order-server`中新增一个接口。

```
// name指定FeignClient的名称，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现
@FeignClient(name = "product-service")
public interface ProductFeignApi {
  @RequestMapping("/product")
  Product findById(@RequestParam("productId") Long productId);
}
```

### 修改Controller

```java
  @Autowired
  ProductFeignApi productFeignApi;
  @Override
  public Order getById(Long oid, Long pid) {
    Product product = productFeignApi.findById(pid);
    Order order = orderDao.getOne(oid);
    order.setPname(product.getPname());
    return order;
  }
```

### Feign的重要属性

我们可以配置超时属性。

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
```

## Feign实现原理

1. 启动后会会根据配置在启动类上的@SpringBootApplication去扫描贴了@FeignClient注解的类，并为其创建代理对象。
2. 通过反射拿到代理类实现的接口：ProductFeignApi。
3. 通过反射拿到接口上注解并且将注解中心的name属性拿出来：product-service。
4. 通过反射拿到接口中的方法，并且拿到接口中方法上的注解@RequestMapping，并且把值拿出来：/product。
5. 将方法中参数注解中的值也拿出来，这个是我们传进来的参数：productId。
6. 拼接出路径：`http://product-service/product?productId=1。`
7. 根据本地的服务清单去找对应的节点信息。
8. 根据你配置的ribbon负载均衡策略去选择节点。
9. 将product-service替换成对应的节点信息和端口。
10. 使用RestTemplate发送请求。

 

 

 

 