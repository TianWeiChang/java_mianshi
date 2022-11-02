Spring Cloud Alibaba 全网最全之Nacos

## 1、服务治理概述

服务治理是微服务架构中最核心最基本的模块。用于实现各个微服务的**自动化注册与发现**。

1. 服务注册：在服务治理框架中，都会构建一个注册中心，每个服务单元向注册中心登记自己提供服务的详细信息。并在注册中心形成一张服务的清单，服务注册中心需要以心跳的方式去监测清单中的服务是否可用，如果不可用，需要在服务清单中剔除不可用的服务。
2. 服务发现：服务调用方向服务注册中心咨询服务，并获取所有服务的实例清单，实现对具体服务实例的访问。

## 2、注册中心的原理

1. 我们再启动的时候，就会把服务的信息告诉注册中心，同时拉去一份最新的1服务列表信息在本地。
2. 每隔一段时间就会去给注册中心发送一个心跳包，告诉注册中心我还活着，同时会拉去一份最新的服务列表信息。
3. 如果这个服务挂了，那么他就不会再给注册中心发送心跳包了。注册中心发现连续几次都没有发送心跳包，那么注册中心就会将这个服务的节点信息给剔除掉，从而1实现动态的注册和踢出服务，就不需要我们手动管理。

![image-20201029084800199](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4960017f2198489ebc8ddf09e5e1a6a1~tplv-k3u1fbpfcp-watermark.awebp)

服务注册中心是微服务架构中1一个十分重要的组件，在微服务架构中11起到了一个协调者的作用，注册中心一般包含这几个功能：

1. 服务发现：
   - 服务注册：保存服务提供者和服务调用者的信息。
   - 服务订阅：服务调用这订阅服务提供者的信息，注册中心向订阅者推送提供者的信息。
2. 服务配置：
   - 配置订阅：服务提供者和服务调用者订阅微服务相关的配置。
   - 配置下发：主动将配置推送给服务提供者和服务调用者。
3. 服务健康检测：检测服务提供者的健康情况，如果发现服务连续几次都没有发送心跳包，说明这个服务有异常，执行服务剔除。

## 3、常见的注册中心

### 3.1、Zookeeper

Zookeeper是一个分布式服务框架，是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。

### 3.2、Eureka

Eureka是Springcloud Netflflix中的重要组件，主要作用就是做服务注册和发现。但是现在已经闭源

### 3.3、Consul

Consul是基于GO语言开发的开源工具，主要面向分布式，服务化的系统提供服务注册、服务发现和配置管理的功能。Consul的功能都很实用，其中包括：服务注册/发现、健康检查、Key/Value存储、多数据中心和分布式一致性保证等特性。Consul本身只是一个二进制的可执行文件，所以安装和部署都非常简单，只需要从官网下载后，在执行对应的启动脚本即可。

### 3.4、Nacos

Nacos是一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。它是 SpringCloud Alibaba 组件之一，负责服务注册发现和服务配置。

## 4、Nacos简介

![image-20210505183124825](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd2d44e3acc14083bde7e28a74418823~tplv-k3u1fbpfcp-watermark.awebp)

Nacos是阿里巴巴2018年7月推出来的一个开源项目，是一个更易于构建云原生应用的动态服务注册与发现、配置管理和服务管理平台。Nacos致力于快速实现动态服务注册与发现、服务配置、服务元数据及流量管理。

他的核心功能：

服务注册：

Nacos Client会通过发送REST请求想Nacos Server注册自己的服务，提供自身的元数据，比如IP地址，端口等信息。Nacos Server接收到注册请求后，就会把这些元数据存储到一个双层的内存Map中。 2. 服务心跳：

在服务注册后，Nacos Client会维护一个定时心跳来维持统治Nacos Server,说明服务一致处于可用状态，防止被剔除，默认5s发送一次心跳。 3. 服务同步：

Nacos Server集群之间会相互同步服务实例，用来保证服务信息的一致性。 4. 服务发现：

服务消费者(Nacos Client)在调用服务提供的服务时，会发送一个REST请求给Nacos Server,获取上面注册的服务清单，并且缓存在Nacos Client本地,同时会在Nacos Client本地开启一个定时任务拉取服务最新的注册表信息更新到本地缓存。 5. 服务健康检查：

 Nacos Server 会开启一个定时任务来检查注册服务实例的健康情况，对于超过15s没有收到客户端心跳的实例会将他的healthy属性设置为false(客户端服务发现时不会发现)，如果某个实例超过30s没有收到心跳，直接剔除该实例(被剔除的实例如果恢复发送心跳则会重新注册)。

## 5、Nacos实战

### 5.1、搭建Nacos环境

#### 5.1.1、下载Nacos环境

我们需要去[下载地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Falibaba%2Fnacos%2Freleases)下载Nacos，下载的Nacos是zip格式的安装包，然后进行解压缩操作。如果在Linux的话需要执行：

```
tar -zxvf Nacos下载包的名称
```

#### 5.1.2、启动

​    我们以Windows下为例，Linux下也是一样的。我们直接切换到bin目录下，然后cmd，输入命令：

```
startup.cmd -m standalone
```

**单机环境必须带-m standalone参数启动，否则无法启动，不带参数启动的是集群环境**。

我们不带参数（以集群方式启动），会出现下列错误。

![image-20210505184150833](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3dc18b7a915d42f49222dfb0651a4c13~tplv-k3u1fbpfcp-watermark.awebp)

正确启动：

![image-20210505184235392](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/24c56b6c343d4bf794c0d0102a0256d7~tplv-k3u1fbpfcp-watermark.awebp)

#### 5.1.3、测试

打开浏览器输入[http://localhost:8848/nacos/index.html#/login](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8848%2Fnacos%2Findex.html%23%2Flogin) ，即可访问服务，默认账号密码都是nacos。

![image-20210505184636822](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02aec4f063e14bd7bede9ea0506b052e~tplv-k3u1fbpfcp-watermark.awebp)

### 5.2、将商品服务注册到Nacos

#### 5.2.1、添加依赖

我们需要在shop-product-server模块中的pom.xml文件中添加nacos的依赖。

```
<!--nacos客户端-->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

#### 5.2.2、在主类上添加注解

```
@SpringBootApplication
@EnableDiscoveryClient
public class ShopProductServerApp {
  public static void main(String[] args) {
    SpringApplication.run(ShopProductServerApp.class,args);
  }
}
```

#### 5.2.3、添加Nacos服务地址

​    我们需要在application.yml中添加nacos服务的地址。

```
spring:
  cloud: 
    nacos: 
      discovery: 
        server-addr: localhost:8848
```

#### 5.2.4、查看

![image-20210505185212140](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18866b1043234092a5d8fe025a13e121~tplv-k3u1fbpfcp-watermark.awebp)

说明注册成功！

### 5.3、将订单服务注册到Nacos

接下来开始修改 shop-order-server 模块的代码， 将其注册到nacos服务上

#### 5.3.1、添加依赖

```
<!--nacos客户端-->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

#### 5.3.2、添加注解

```
@SpringBootApplication
@EnableDiscoveryClient
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

#### 5.3.3、添加Nacos服务地址

```
spring:
  cloud: 
    nacos: 
      discovery: 
        server-addr: localhost:8848
```

#### 5.3.4、测试

![image-20210505185547641](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da1e16a635ec47a1b7cc873538bd33e7~tplv-k3u1fbpfcp-watermark.awebp)

### 5.4、编写测试代码

```
  @Autowired
  private DiscoveryClient discoveryClient;
  @RequestMapping("test")
  public String test(){
    ServiceInstance serviceInstance = discoveryClient.getInstances("product-service").get(0);
    return serviceInstance.toString();
  }
```

![image-20210505191627907](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3795b0ae24ed4e96a14a55919d3d9362~tplv-k3u1fbpfcp-watermark.awebp)

 

 

 

 