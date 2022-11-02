面试官：了解微服务组件Feign的工作原理吗？

## Feign的工作原理

主程序入口添加了`@EnableFeignClients`注解开启对`FeignClient`扫描加载处理。根据Feign Client的开发规范，定义接口并加`@FeignClientd`注解。

当程序启动时，会进行包扫描，扫描所有@FeignClients的注解的类，并且将这些信息注入Spring IOC容器中，当定义的的Feign接口中的方法被调用时，通过JDK的代理方式，来生成具体的RequestTemplate.

当生成代理时，`Feign`会为每个接口方法创建一个`RequestTemplate`。当生成代理时，Feign会为每个接口方法创建一个`RequestTemplate`对象，该对象封装了HTTP请求需要的全部信息，如请求参数名，请求方法等信息都是在这个过程中确定的。

然后`RequestTemplate`生成Request,然后把Request交给Client去处理，这里指的是`Client`可以是`JDK`原生的`URLConnection`，`Apache`的`HttpClient`,也可以是`OKhttp`，最后Client被封装到`LoadBalanceClient`类，这个类结合Ribbon负载均衡发起服务之间的调用。

## Feign注解剖析

@`FeignClient`注解主要被`@Target({ElementType.TYPE})`修饰，表示该注解主要使用在接口上。它具备了如下的属性：

- name:指定FeignClient的名称，如果使用了Ribbon，name就作为微服务的名称，用于服务发现。
- url:url一般用于调试，可以指定@FeignClient调用的地址。
- decode404: 当发生404错误时，如果该字段为true，会调用decoder进行解码，否则抛出FeignException.
- configuration:Feign配置类，可以自定或者配置Feign的Encoder，Decoder，LogLevel，Contract。
- fallback:定义容错的处理类，当调用远程接口失败或者超时时，会调用对应的接口的容错逻辑，fallback指定的类必须实现@Feign标记的接口。
- fallbacjFactory:工厂类，用于生成fallback类实例，通过这个属性可以实现每个接口通用的容错逻辑们介绍重复的代码。
- path：定义当前FeignClient的统一前缀。

## Feign开启GZIP压缩

`Spring Cloud Feign`支持对请求和响应进行`GZIP`压缩，以提高通信效率。

在`yml`文件需要如下的配置

```
feign:
  compression:
    request:
      enabled: true #开请求压缩
      mimeTypes: #媒体类型 text/xml,application/xml,application/json
      minRequestSize: 2048 #最小的请求大小
    response:
      enabled: true #开启响应的压缩
```

需要注意的是，在采用了压缩之后，需要使用二进制的方式进行数据传递，所有返回值都需要使用 `ResponseEntity<byte[]>` 接收.

```
@FeignClient(name = "github-client",url = "https://api.github.com",configuration = HelloFeignServiceConfig.class)
public interface HelloFeignService {

    /*
    这个返回类型如果采用了压缩，那么就是二进制的方式，就需要使用ResponseEntity<byte[]>作为返回值
     */
    @RequestMapping(value = "/search/repositories",method = RequestMethod.GET)
    ResponseEntity<byte[]> searchRepositories(@RequestParam("q")String parameter);
}
```

## Feign Client开启日志

需要在注解类添加配置的类：

```
@FeignClient(name = "github-client",url = "https://api.github.com",configuration = HelloFeignServiceConfig.class)
```

注解类的代码如下:

```
@Configuration
public class HelloFeignServiceConfig {

    /**
     *
     * Logger.Level 的具体级别如下：
         NONE：不记录任何信息
         BASIC：仅记录请求方法、URL以及响应状态码和执行时间
         HEADERS：除了记录 BASIC级别的信息外，还会记录请求和响应的头信息
         FULL：记录所有请求与响应的明细，包括头信息、请求体、元数据
     * @return
     */
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

}
```

## Feign的超时设置

Feign的调用分两层，即Ribbon层的调用和Hystrix的调用，高版本的Hystrix默认是关闭的。

![图片](https://mmbiz.qpic.cn/mmbiz/8KKrHK5ic6XBDVUibBkDN4ryo8ibQFQhnRyyl6mUEmwaK2mvib2mMXKFHysnPZMVel3052QzA2Z39SGzR33ERncdmA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果出现上述的错误，那么就需要添加如下的配置

```
#请求处理的超时时间
ribbon.ReadTimeout: 12000
#请求链接超时时间
ribbon.ConnectionTimeout: 30000
```

如果开启了Hystrix，Hystrix的超时报错信息如下：

![图片](https://mmbiz.qpic.cn/mmbiz/8KKrHK5ic6XBDVUibBkDN4ryo8ibQFQhnRy9trA1omNqqpCic7nZa7ttVRX22HxZnuyFW4rbvK3j7Y70aIZURHeKlw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

此时可以添加如下配置:

```
hystrix:
  command:
    default:
      circuitBreaker:
        sleepWindowInMilliseconds: 30000
        requestVolumeThreshold: 50
      execution:
        timeout:
          enabled: true
        isolation:
          strategy: SEMAPHORE
          semaphore:
            maxConcurrentRequests: 50
          thread:
            timeoutInMilliseconds: 100000
```

## Feign的Post和Get的多参数传递

在`Spring MVC`中可以使用Post或者Get轻易的请求到对应的接口并且将参数绑定到`POJO`，但是`Feign`并没有全部实现`Spring MVC`的功能，如果使用GET请求到接口，就无法将参数绑定到`POJO`，但是可以使用以下的几种方式实现相应的功能。

- 将POJO内的字段作为一个个的字段写在接口上
- 将参数编成一个map
- `POJO`使用`@RequestBody`修饰，但是这个违反`Restful`的原则

以下的代码将Feign的请求拦截下来，将参数进行处理之后统一的编成map

```
@Component
public class FeignRequestInterceptor implements RequestInterceptor{
    @Autowired
    private ObjectMapper objectMapper;



    @Override
    public void apply(RequestTemplate requestTemplate) {
        // feign 不支持 GET 方法传 POJO, json body转query
        if (requestTemplate.method().equalsIgnoreCase("GET") && requestTemplate.body()!=null){
            try {
                JsonNode jsonNode = objectMapper.readTree(requestTemplate.body());
                requestTemplate.body(null);
                Map<String,Collection<String>> queries = new HashMap<>();
                buildQuery(jsonNode,"",queries);
                requestTemplate.queries(queries);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }

    }
    private void buildQuery(JsonNode jsonNode, String path, Map<String, Collection<String>> queries) {
        if (!jsonNode.isContainerNode()) {   // 叶子节点
            if (jsonNode.isNull()) {
                return;
            }
            Collection<String> values = queries.get(path);
            if (null == values) {
                values = new ArrayList<>();
                queries.put(path, values);
            }
            values.add(jsonNode.asText());
            return;
        }
        if (jsonNode.isArray()) {   // 数组节点
            Iterator<JsonNode> it = jsonNode.elements();
            while (it.hasNext()) {
                buildQuery(it.next(), path, queries);
            }
        } else {
            Iterator<Map.Entry<String, JsonNode>> it = jsonNode.fields();
            while (it.hasNext()) {
                Map.Entry<String, JsonNode> entry = it.next();
                if (StringUtils.hasText(path)) {
                    buildQuery(entry.getValue(), path + "." + entry.getKey(), queries);
                } else {  // 根节点
                    buildQuery(entry.getValue(), entry.getKey(), queries);
                }
            }
        }
    }

}
```

除了上面的方法之外还可以使用`venus-cloud-feign`这个依赖，该依赖实现了GET请求的参数Map包装，无需再去自己写。依赖如下:

```
<!-- https://mvnrepository.com/artifact/cn.springcloud.feign/venus-cloud-feign-core -->
<dependency>
    <groupId>cn.springcloud.feign</groupId>
    <artifactId>venus-cloud-feign-core</artifactId>
    <version>1.0.0</version>
</dependency>
```

github的地址如下:

> https://github.com/SpringCloud/venus-cloud-feign.git

asdas