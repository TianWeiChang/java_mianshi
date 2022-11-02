为什么要使用注册中心？是`Eureka`还是`Nacos`？

## 为什么要使用注册中心

有使用过`ip:port`地址直接调用服务的开发经历么？该段痛苦的经历在此处省略500字......，该种方式的缺点： 

- 需要手动的维护所有的服务访问ip地址列表。 
- 单个服务实现负载均衡需要自己搭建，例如使用nginx负载均衡策略，或者基于容器化多实例部署单个服务，在实例之间做负载均衡。 

使用注册中心能够实现服务治理，服务动态扩容，以及服务调用的负载均衡，完整调用链路示例如下： 

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpT3ca695ibbbT9kwJianD8uZhUhMEvO8VTDjrMUbra5SKGM6icXo8aOk3icicoYXGdibOpdl5y5qibFibSpGg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- **服务提供者**：向注册中心根据服务名称提供服务访问的`ip:port`以及其他信息。
- **注册中心**：根据服务名称，存储对应的`ip:port`以及其他信息。
- **服务消费者**：根据服务名向注册中心获取调用服务的`ip:port`以及其他相关的信息集合，然后根据负载均衡策略获取最终的服务器`ip:port`访问地址。

> 使用`Spring Cloud`时，常用的是`Eureka`和`Nacos`作为注册中心，如何选择呢？

推荐一个 `Spring Boot `基础教程及实战示例：`https://github.com/javastacks/spring-boot-best-practice`

## Eureka注册中心

架构原理图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpT3ca695ibbbT9kwJianD8uZhC4MUxNwhjC6TF1qD6Q9Z7YciabA6uAX4gCUdWPtNkwgGHSVobdnsSPA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**服务提供者**

主动向注册中心注册，续约，下线，获取注册表。服务注册成功后，定时向注册中心发送心跳，保证服务不被剔除。

**注册中心**

存储服务实例，定时扫描注册表，剔除过期的服务实例。通过同步复制方式实现高可用，先获取注册表，然后再向其他注册中心注册自己，属于AP模式。

在实际项目中，会根据环境，例如`dev`、`test`、`prod`配置不同的注册中心集群，如果不同的项目使用统一的注册中心，只能根据服务名称做区分。

重点介绍一下Eureka自我保护机制。如果出现大量的服务实例过期被剔除，则注册中心进入自我保护模式，注册表中信息不再被剔除，目的是提高eureka的可用性。默认情况下，统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些实例不会过期。

讲述一次惨痛的上线经历，错误描述如下：

> 当时服务部署成功，在Eureka注册中心已经显示该服务已经注册成功，但是，前端请求经过网关再转发到该服务时，一直就没有反应，服务调用一直不成功。`Nginx`转发，网关转发都在确认问题到底发生在哪里，几经折磨，在网关直接通过`IP`地址转发到上线的服务，快速的解决该问题。
>
> 后续，复盘，应该Eureka的自我保护机制，导致的问题。在注册中心注册的服务是一个不可用的服务，但是，由于自我保护机制，Eureka Server没有将无效的服务剔除。

后续的解决方法是，设置`enableSelfPreservation=false`关闭自我保护机制，把renewalPercentThreshold 比例降低，在Eureka Server端，如果出现无效的服务就会将该服务剔除。 

## Nacos注册中心

`Nacos`是`Spring Cloud`的扩展，注册中心功能通过`NacosDiscoveryClient` 继承`DiscoveryClient`，在`Spring Cloud`中，与Eureka可以无侵入的切换。

注册中心可以手动剔除服务实例，通过消息通知客户端更新缓存的实例信息，完整调用链路示例如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpT3ca695ibbbT9kwJianD8uZhy6pGkEaMTlb4HcfS7AmEOJsdXqQrjsIpVTRAiaI2hjsQ8zGZ8UhdaSA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在`Spring Cloud`中引入`Nacos`时，参考官网匹配具体的版本，如图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpT3ca695ibbbT9kwJianD8uZho1rNypicc42Loz9PY7rxpibsnZ0A4Kvx93M4rKUgltkZEk7icQUxuzxtw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`Nacos`重点需要了解下其领域模型Nacos 数据模型 Key 由三元组唯一确定, Namespace命名空间，分组group，service服务。详情可以参考官网Nacos 架构。

**Nacos与Eureka相比优势如下：**

- `Nacos`在自动或手动下线服务，使用消息机制通知客户端，服务实例的修改很快响应；Eureka只能通过任务定时剔除无效的服务。 
- `Nacos`可以根据namespace命名空间，DataId,Group分组，来区分不同环境（dev，test，prod），不同项目的配置。 

SFS