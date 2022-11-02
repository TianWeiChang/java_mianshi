# Spring Cloud Alibaba 全网最全讲解

## Spring Cloud

### 1、什么是Spring Cloud

Spring Cloud是一个含概多个子项目的开发工具集,集合了众多的开源框架，他利用了Spring Boot开发的便利性实现了很多功能。如服务注册、服务发现、负载均衡等，Spring Cloud在整合过程中主要是针对Netflix(奈飞)开源组件的封装。Spring Cloud的出现真正的简化了分布式架构的开发。

NetFlix 是美国的一个在线视频网站,微服务业的翘楚,他是公认的大规模生产级微服务的杰出实践者，NetFlix的开源组件已经在他大规模分布式微服务环境中经过多年的生产实战验证,因此Spring Cloud中很多组件都是基于NetFlix组件的封装。

### 2、核心组件

1. eurekaserver、consul、nacos：服务注册中心组件。
2. rabbion & openfeign：服务负载均衡 和 服务调用组件。
3. hystrix & hystrix dashboard：服务断路器和服务监控组件。
4. zuul、gateway：服务网关组件。
5. config：统一配置中心组件。
6. bus：消息总线组件。

![image-20200724161314786](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a879750c9de41fc8f9acd90c27a3b17~tplv-k3u1fbpfcp-watermark.awebp)

### 3、版本命名

SpringCloud是一个由众多独立子项目组成的大型综合项目，原则每个子项目上有不同的发布节奏,都维护自己发布版本号。为了更好的管理springcloud的版本,通过一个资源清单BOM(Bill of Materials),为避免与子项目的发布号混淆，所以没有采用版本号的方式，而是通过命名的方式。这些名字是按字母顺序排列的。如伦敦地铁站的名称（“天使”是第一个版本，“布里斯顿”是第二个版本,"卡姆登"是第三个版本）。当单个项目的点发布累积到一个临界量，或者其中一个项目中有一个关键缺陷需要每个人都可以使用时，发布序列将推出名称以“.SRX”结尾的“服务发布”，其中“X”是一个数字。

伦敦地铁站的名字大致有如下：Angel、Brixton、Camden、Dalston、Edgware、Finchley、Greenwich、Hoxton。

### 4、版本选择

由于SpringCloud的版本是必须和SpringBoot的版本对应的，所以必须要根据SpringCloud版本来选择SpringBoot的版本。

![image-20200709112427684](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/694814e4456049118dc39e4c47e45558~tplv-k3u1fbpfcp-watermark.awebp)

## SpringCloud Alibaba

### 1、简介

Spring Cloud Alibaba是Spring Cloud下的一个子项目，Spring Cloud Alibaba为分布式应用程序开发提供了一站式解决方案，它包含开发分布式应用程序所需的所有组件，使您可以轻松地使用Spring Cloud开发应用程序，使用Spring Cloud Alibaba，您只需要添加一些注解和少量配置即可将Spring Cloud应用程序连接到Alibaba的分布式解决方案，并使用Alibaba中间件构建分布式应用程序系统。Spring Cloud Alibaba 是阿里巴巴开源中间件跟 Spring Cloud 体系的融合。

![image-20210504210759006](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ceed2826ad24a84ab6eb41a433b3d9a~tplv-k3u1fbpfcp-watermark.awebp)

### 2、主要功能

1. 流量控制和服务降级：默认支持 WebServlet、WebFlux， OpenFeign、RestTemplate、Spring Cloud、Gateway， Zuul， Dubbo 和 RocketMQ 限流降级功能的接入，可以在运行时通过控制台实时修改限流降级规则，还支持查看限流降级 Metrics 监控。
2. 服务注册和发现：实例可以在Alibaba Nacos上注册，客户可以使用Spring管理的bean发现实例，通过Spring Cloud Netflix支持Ribbon客户端负载均衡器。
3. 分布式配置管理：支持分布式系统中的外部化配置，配置更改时自动刷新。
4. 消息驱动能力：基于 Spring Cloud Stream 为微服务应用构建消息驱动能力。
5. 消息总线：使用Spring Cloud Bus RocketMQ链接分布式系统的节点。
6. 分布式事务：使用 @GlobalTransactional 注解， 高效并且对业务零侵入地解决分布式事务问题。
7. Dubbo RPC：通过Apache Dubbo RPC扩展Spring Cloud服务到服务调用的通信协议。
8. 分布式任务调度：提供秒级、精准、高可靠、高可用的定时（基于 Cron 表达式）任务调度服务。同时提供分布式的任务执行模型，如网格任务。网格任务支持海量子任务均匀分配到所有Worker（schedulerx-client）上执行。

### 3、组件

1. Sentinel：把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。
2. Nacos：一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。
3. RocketMQ：一款开源的分布式消息系统，基于高可用分布式集群技术，提供低延时的、高可靠 的消息发布与订阅服务。
4. Dubbo：Apache Dubbo™ 是一款高性能 Java RPC 框架。
5. Seata：阿里巴巴开源产品，一个易于使用的高性能微服务分布式事务解决方案。
6. Alibaba Cloud ACM：一款在分布式架构环境中对应用配置进行集中管理和推送的应用配置中心 产品。
7. Alibaba Cloud OSS: 阿里云对象存储服务（Object Storage Service，简称 OSS），是阿里云提 供的海量、安全、低成本、高可靠的云存储服务。您可以在任何应用、任何时间、任何地点存储和 访问任意类型的数据。
8. Alibaba Cloud SchedulerX: 阿里中间件团队开发的一款分布式任务调度产品，提供秒级、精 准、高可靠、高可用的定时（基于 Cron 表达式）任务调度服务。
9. Alibaba Cloud SMS: 覆盖全球的短信服务，友好、高效、智能的互联化通讯能力，帮助企业迅速 搭建客户触达通道。

![image-20210504211009062](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/652add8f8f7a4e4dbd6ce6ae7d6f0e39~tplv-k3u1fbpfcp-watermark.awebp)

## 微服务项目搭建

### 1、技术选型

- 持久层：SpingData Jpa
- 数据库: MySQL5.7
- 技术栈：SpringCloud Alibaba 技术栈

### 2、模块设计

我们搭建一个微服务的项目，但是只有简单的代码，没有任何业务逻辑。

- shop-parent 父工程
- shop-product-api：商品微服务api ，用于存放商品实体。
- shop-product-server：商品微服务，他的1端口是808x。
- shop-order-api 订单微服务api，用于存放订单实体。
- shop-order-server 订单微服务，他的端口是808x。

### 3、微服务的调用

 在微服务架构中，最常见的场景就是微服务之间的相互调用。我们以电商系统中常见的**用户下单**为例来演示微服务的调用：客户向订单微服务发起一个下单的请求，在进行保存订单之前需要调用商品微服务查询商品的信息。

我们一般把服务的主动调用方称为**服务消费者**，把服务的被调用方称为**服务提供者**。

![image-20201028144911439](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee97f8f963f94340a0cfd7803bb114d0~tplv-k3u1fbpfcp-watermark.awebp)

### 4、创建父工程

创建一个maven工程，然后在pom.xml文件中添加下面内容

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.3.RELEASE</version>
    <relativePath/>
  </parent>
  <groupId>cn.linstudy</groupId>
  <artifactId>Shop-parent</artifactId>
  <packaging>pom</packaging>
  <version>1.0.0</version>
  <modules>
    <module>Shop-order-api</module>
    <module>Shop-order-server</module>
    <module>Shop-product-api</module>
    <module>Shop-product-server</module>
  </modules>
  <properties>
    <java.version>11</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <spring-cloud.version>Hoxton.SR8</spring-cloud.version>
    <spring-cloud-alibaba.version>2.2.3.RELEASE</spring-cloud-alibaba.version>
  </properties>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-dependencies</artifactId>
        <version>${spring-cloud-alibaba.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.47</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

### 5、创建商品服务

#### 书写Shop-product-api的依赖

创建Shop-product-api项目，然后在pom.xml文件中添加下面内容

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
  <artifactId>Shop-product-api</artifactId>

  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>
  </dependencies>
</project>
```

#### 创建实体

```
//商品
@Entity(name = "t_shop_product")
@Data
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long pid;//主键
    private String pname;//商品名称
    private Double pprice;//商品价格
    private Integer stock;//库存
}
```

#### 书写Shop-product-server的依赖

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

  <artifactId>Shop-order-server</artifactId>

  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>fastjson</artifactId>
      <version>1.2.56</version>
    </dependency>
    <dependency>
      <groupId>cn.linstudy</groupId>
      <artifactId>Shop-order-api</artifactId>
      <version>1.0.0</version>
    </dependency>
    <dependency>
      <groupId>cn.linstudy</groupId>
      <artifactId>Shop-product-api</artifactId>
      <version>1.0.0</version>
    </dependency>
      
   </dependencies>
</project>
```

#### 编写application.yml

```
server:
  port: 8081
spring:
  application:
    name: product-service
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql:///shop-product?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: admin
  jpa:
    properties:
      hibernate:
        hbm2ddl:
          auto: update
        dialect: org.hibernate.dialect.MySQL5InnoDBDialect
```

#### 创建数据库

由于我们使用的是JPA，所以我们需要创建数据库，但是不需要创建表，因为JPA会在对应的数据库中自动创建表。

#### 创建DAO接口

```
// 第一个参数是实体类，第二个参数是实体类对象的主键的类型
public interface ProductDao extends JpaRepository<Product,Long> {

}
```

#### 创建Service接口及其实现类

```
public interface ProductService {
  Product findById(Long productId);
}
```

```
@Service
public class ProductServiceImpl implements ProductService {
  @Autowired
  ProductDao productDao;
    
  @Override
  public Product findById(Long productId) {
    return productDao.findById(productId).get();
  }
}
```

#### 书写Controller

```
@RestController
@Slf4j
public class ProductController {
    @Autowired
    private ProductService productService;
    
    //商品信息查询
    @RequestMapping("/product")
    public Product findByPid(@RequestParam("pid") Long pid) {
             Product product = productService.findByPid(pid);
             return product;
    }
}
```

#### 加入测试数据

我们在启动项目的时候，可以发现表已经默认帮我们自动创建好了，我们需要导入测试数据。

```
INSERT INTO t_shop_product VALUE(NULL,'小米','1000','5000'); 
INSERT INTO t_shop_product VALUE(NULL,'华为','2000','5000'); 
INSERT INTO t_shop_product VALUE(NULL,'苹果','3000','5000'); 
INSERT INTO t_shop_product VALUE(NULL,'OPPO','4000','5000');
```

#### 测试

![image-20210505140802977](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b195cad5842496bb269cb5c3d023ca6~tplv-k3u1fbpfcp-watermark.awebp)

### 6、创建订单微服务

#### 书写Shop-order-api的依赖

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

  <artifactId>Shop-order-api</artifactId>

  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-annotations</artifactId>
    </dependency>
  </dependencies>
</project>
```

#### 创建订单实体

```
//订单
@Entity(name = "t_shop_order")
@Data
@JsonIgnoreProperties(value = { "hibernateLazyInitializer"})
public class Order implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long oid;//订单id

    //用户
    private Long uid;//用户id
    private String username;//用户名
    //商品
    private Long pid;//商品id
    private String pname;//商品名称
    private Double pprice;//商品单价
    //数量
    private Integer number;//购买数量
}
```

#### 创建shop-order-server项目

在pom.xml中添加依赖。

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

  <artifactId>Shop-order-server</artifactId>

  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>fastjson</artifactId>
      <version>1.2.56</version>
    </dependency>
    <dependency>
      <groupId>cn.linstudy</groupId>
      <artifactId>Shop-order-api</artifactId>
      <version>1.0.0</version>
    </dependency>
    <dependency>
      <groupId>cn.linstudy</groupId>
      <artifactId>Shop-product-api</artifactId>
      <version>1.0.0</version>
    </dependency>
</project>
```

#### 编写application.yml

```
server:
  port: 8082
spring:
  application:
    name: order-service
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql:///shop-product?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: 1101121833
  jpa:
    properties:
      hibernate:
        hbm2ddl:
          auto: update
        dialect: org.hibernate.dialect.MySQL5InnoDBDialect
```

#### 编写Service及其实现类

​    我们写的Service接口及其实现类不具备任何的业务逻辑，仅仅只是为了测试而用。

```
public interface OrderService {
  Order getById(Long oid,Long pid);
}
```

```
@Service
public class OrderServiceImpl implements OrderService {

  @Autowired
  OrderDao orderDao;

  @Override
  public Order getById(Long oid, Long pid) {
    return orderDao.getOne(oid);
  }
}
```

#### 创建Controller

```
@RestController
@Slf4j
public class OrderController {
    @Autowired
    private OrderService orderService;
  	@RequestMapping("getById")
  	public Order getById(Long oid,Long pid){
        return orderService.getById(oid, pid);
}
```

### 7、服务之间如何进行调用

假设我们在订单的服务里面需要调用到商品服务，先查询出id为1的商品，然后再查询出他的订单，这个1时候就涉及到服务之间的调用问题了。     服务之间的1调用本质上是通过Java代码去发起一个Http请求，我们可以使用RestTemplate来进行调用。

#### 在shop-order-server的启动类中添加注解

 我们既然需要RestTemplate类，就需要将RestTemplate类注入到容器中。

```
@SpringBootApplication
public class OrderServer {
    public static void main(String[] args) {
        SpringApplication.run(OrderServer.class,args);
    }
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

#### 修改Controller代码

```
@RestController
public class OrderController {

  @Autowired
  RestTemplate restTemplate;

  @Autowired
  OrderService orderService;
  @RequestMapping("getById")
  public Order getById(Long oid,Long pid){
    Product product = restTemplate.getForObject(
        "http://localhost:8083/product?productId="+pid,Product.class); // 这里通过restTemplate去发起http请求，请求地址是http://localhost:8083/product，携带的参数是productId
    Order order = orderService.getById(oid, product.getPid());
    order.setUsername(product.getPname());
    return order;
  }
}
```

这样我们就完成了服务之间的互相调用。

### 存在的问题

虽然我们已经可以实现微服务之间的调用。但是我们把服务提供者的网络地址（ip，端口）等硬编码到了代码中，这种做法存在许多问题：

- 一旦服务提供者地址变化，就需要手工修改代码。
- 一旦是多个服务提供者，无法实现负载均衡功能。
- 一旦服务变得越来越多，人工维护调用关系困难。

那么应该怎么解决呢?这时候就需要通过**注册中心**动态的实现**服务治理**。



 

 

 

 