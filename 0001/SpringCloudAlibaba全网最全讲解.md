Spring Cloud Alibaba 全网最全讲解

## 1、微服务简介

> In short, the microservice architectural style is an approach to developing a single application as **a suite of small services**, each **running in its own process** and communicating with lightweight mechanisms, often an HTTP resource API. These services are **built around business capabilities** and **independently deployable** by fully automated deployment machinery. **There is a bare minimum of centralized management of these services**, which may be written in different programming languages and use different data storage technologies.                   -----[摘自官网]    

简而言之，微服务架构风格是一种将单个应用程序开发为“一套小型服务”的方法，每个服务“运行在自己的进程中”，并通过轻量级机制(通常是HTTP资源API)进行通信。这些服务“围绕业务功能构建”，并通过全自动部署机制“独立部署”。“这些服务只有最低限度的集中管理”，可能是用不同的编程语言编写的，并使用不同的数据存储技术。

**微服务是一种架构，这种架构是将单个的整体应用程序分割成更小的项目关联的独立的服务。一个服务通常实现一组独立的特性或功能，包含自己的业务逻辑和适配器。各个微服务之间的关联通过暴露api来实现。这些独立的微服务不需要部署在同一个虚拟机，同一个系统和同一个应用服务器中。**

## 2、为什么是微服务

### 2.1、单体应用

一个系统业务量很小的时候所有的代码都放在**一个项目**中就好了，然后这个项目部署在一台服务器上就好了。整个项目所有的服务都由这台服务器提供。这就是单机结构。

![image-20200708224716035](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ccbeaf3186e44388877f8fc54cef626f~tplv-k3u1fbpfcp-watermark.awebp)

单体应用的优点在于：单一架构模式在项目初期很小的时候开发方便，测试方便，部署方便，运行良好。

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7a4a94c10b64fe5bdb1ed450982a508~tplv-k3u1fbpfcp-watermark.awebp)

他的缺点也很明显：

1. 应用随着时间的推进，加入的功能越来越多，最终会变得巨大，一个项目中很有可能数百万行的代码，互相之间繁琐的jar包。
2. 久而久之，开发效率低，代码维护困难。
3. 如果想整体应用采用新的技术，新的框架或者语言，那是不可能的。
4. 任意模块的漏洞或者错误都会影响这个应用，降低系统的可靠性。

### 2.2、分布式

由于整个系统运行需要使用到Tomcat和MySQL，单台服务器处理的能力有限,2G的内存需要分配给Tomcat和MySQL使用，随着业务越来越复杂，请求越来越多.。内存越来越不够用了，所以这时候我们就需要进行分布式的部署。

我们进行一个评论的请求，这个请求是需要依赖**分布**在两台不同的服务器的组件[Tomat和MySQL],才能完成的.。所以叫做分布式的系统。

![image-20201027173909529](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/774a090935a14dde9a9913cfb538e745~tplv-k3u1fbpfcp-watermark.awebp)

分布式和单体项目最大的区别在于分布式的项目是分开部署的，比如说把数据库单独放在一台服务器上。

### 2.3、集群

在上面的图解中其实是存在问题的，比如Tomcat存在单点故障问题，一旦Tomcat所在的服务器宕机不可用了，我们就无法提供服务了,所以针对单点故障问题，我们会使用集群来解决.那什么是集群模式呢

单机处理到达瓶颈的时候，你就把单机复制几份，这样就构成了一个“集群”。集群中每台服务器就叫做这个集群的一个“节点”，所有节点构成了一个集群。每个节点都提供相同的服务，那么这样系统的处理能力就相当于提升了好几倍（有几个节点就相当于提升了这么多倍）。

但问题是用户的请求究竟由哪个节点来处理呢？最好能够让此时此刻负载较小的节点来处理，这样使得每个节点的压力都比较平均。要实现这个功能，就需要在所有节点之前增加一个“调度者”的角色，用户的所有请求都先交给它，然后它根据当前所有节点的负载情况，决定将这个请求交给哪个节点处理。这个“调度者”有个牛逼了名字——负载均衡服务器。

![image-20201027182534219](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f03bea8095e84b7daa5aed0ca9f0c9f9~tplv-k3u1fbpfcp-watermark.awebp)

## 3、系统架构的演变

架构的演变大致为：单一应用架构 `===>` 垂直应用架构 `===>` 分布式服务架构 `===>` 流动计算架构微服务架构 `===>` [未知]

### 3.1、单一应用架构

互联网早期，一般的网站应用流量较小，只需一个应用，将所有功能代码都部署在一起就可以，这样可以减少开发、部署和维护的成本。

比如说一个电商系统，里面会包含很多用户管理，商品管理，订单管理，物流管理等等很多模块，

我们会把它们做成一个web项目，然后部署到一台tomcat服务器上。

![image-20210506104017112](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34e59ae1ad604a08b94da75921153b32~tplv-k3u1fbpfcp-watermark.awebp)

他的优点在于：

1. 项目架构简单，小型项目的话， 开发成本低。
2. 项目部署在一个节点上，维护容易。

他的缺点也是显而易见的：

- 全部功能集成在一个工程中，对于大型项目来讲不易开发和维护。
- 项目模块之间紧密耦合，单点容错率低。
- 无法针对不同模块进行针对性优化和水平扩展。

### 3.2、垂直应用架构

随着访问量的逐渐增大，单一应用只能依靠增加节点来应对，但是这时候会发现并不是所有的模块都会有比较大的访问量.

还是以上面的电商为例子， 用户访问量的增加可能影响的只是用户和订单模块， 但是对消息模块的影响就比较小. 那么此时我们希望只多增加几个订单模块， 而不增加消息模块. 此时单体应用就做不到了， 垂直应用就应运而生了.

所谓的垂直应用架构，就是将原来的一个应用拆成互不相干的几个应用，以提升效率。比如我们可以将上面电商的单体应用拆分成:

- 电商系统(用户管理 商品管理 订单管理)
- 后台系统(用户管理 订单管理 客户管理)
- CMS系统(广告管理 营销管理)

这样拆分完毕之后，一旦用户访问量变大，只需要增加电商系统的节点就可以了，而无需增加后台和CMS的节点。

![image-20210506104056378](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9597bdb23b543439ad01dc64931c51b~tplv-k3u1fbpfcp-watermark.awebp)

他的优点在于：

1. 系统拆分实现了流量分担，解决了并发问题，而且可以针对不同模块进行优化和水平扩展。
2. 一个系统的问题不会影响到其他系统，提高容错率。

缺点：

1. 系统之间相互独立， 无法进行相互调用。
2. 系统之间相互独立， 会有重复的开发任务

### 3.3、分布式架构

当垂直应用越来越多，重复的业务代码就会越来越多。这时候，我们就思考可不可以将重复的代码抽取出来，做成统一的业务层作为独立的服务，然后由前端控制层调用不同的业务层服务呢？     这就产生了新的分布式系统架构。它将把工程拆分成表现层和服务层两个部分，服务层中包含业务逻辑。表现层只需要处理和页面的交互，业务逻辑都是调用服务层的服务来实现。

![image-20210506104118727](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34bca4eb4b4a4ab1ad1570d85c4e1789~tplv-k3u1fbpfcp-watermark.awebp)

优点：

1. 抽取公共的功能为服务层，提高代码复用性。

缺点：

1. 系统间耦合度变高，调用关系错综复杂，难以维护。

### 3.4、SOA架构

在分布式架构下，当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心对集群进行实时管理。此时，用于资源调度和治理中心(SOA Service Oriented Architecture，面向服务的架构)是关键。

![image-20210506104351544](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e311caccd84542c487bdbd39e67fa9d5~tplv-k3u1fbpfcp-watermark.awebp)

优点：

1. 使用注册中心解决了服务间调用关系的自动调节

缺点：

1. 服务间会有依赖关系，一旦某个环节出错会影响较大( 服务雪崩 )。
2. 服务关心复杂，运维、测试部署困难。

### 3.5、微服务架构

微服务架构在某种程度上是面向服务的架构，它更加强调服务的"彻底拆分"。

![image-20210506104415430](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3a74b1021b244259b4656db9e05f857~tplv-k3u1fbpfcp-watermark.awebp)

优点：

1. 服务原子化拆分，独立打包、部署和升级，保证每个微服务清晰的任务划分，利于扩展。
2. 微服务之间采用RESTful等轻量级Http协议相互调用。
3. 服务各自有自己单独的职责，服务之间松耦合，避免因一个模块的问题导致服务崩溃

缺点：

1. 分布式系统开发的技术成本高（容错、分布式事务等）。
2. 服务治理和服务监控关键。
3. 多服务运维难度，随着服务的增加，运维的压力也在增大

## 4、微服务需要解决的问题

微服务架构， 简单的说就是将单体应用进一步拆分，拆分成更小的服务，每个服务都是一个可以独立运行的项目。

### 微服务架构的常见问题

一旦采用微服务系统架构，就势必会遇到这样几个问题：

- 这么多小服务，如何管理他们？
- 这么多小服务，他们之间如何通讯？
- 这么多小服务，客户端怎么访问他们？
- 这么多小服务，一旦出现问题了，应该如何自处理？
- 这么多小服务，一旦出现问题了，应该如何排错?

​    对于上面的问题，是任何一个微服务设计者都不能绕过去的，因此大部分的微服务产品都针对每一个问题提供了相应的组件来解决它们。

![image-20210506104432772](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d85ba2d7d04e4aa0b5681aa1250dec2b~tplv-k3u1fbpfcp-watermark.awebp)

## 5、微服务架构的常见概念

### 5.1、服务治理

服务治理就是进行服务的自动化管理，其核心是服务的注册与发现。

- 服务注册：服务实例将自身服务信息注册到注册中心。
- 服务发现：服务实例通过注册中心，获取到注册到其中的服务实例的信息，通过这些信息去请求他们提供服务。
- 服务剔除：服务注册中心将出问题的服务自动剔除到可用列表之外，使其不会被调用到。

![image-20210506103427249](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1496fd4662614e93a7379be5b02b5f14~tplv-k3u1fbpfcp-watermark.awebp)

### 5.2、服务调用

在微服务架构中，通常存在多个服务之间的远程调用的需求，目前1主流的远程调用的技术有基于HTTP请求的RESTFul接口及基于TCP的RPC协议。

- REST(Representational State Transfer)：这是一种HTTP调用的格式，更标准，更通用，无论哪种语言都支持http协议。
- RPC（Remote Promote Call）：一种进程间通信方式。允许像调用本地服务一样调用远程服务。RPC框架的主要目标就是让远程服务调用更简单、透明。RPC框架负责屏蔽底层的传输方式、序列化方式和通信细节。开发人员在使用的时候只需要了解谁在什么位置提供了什么样的远程服务接口即可，并不需要关心底层通信细节和调用过程。

他们之间的区别于联系：

| 比较项   | RESTFul    | RPC       |
| -------- | ---------- | --------- |
| 通讯协议 | HTTP       | 一般是TCP |
| 性能     | 略低       | 较高      |
| 灵活度   | 高         | 低        |
| 应用     | 微服务架构 | SOA架构   |

### 5.3、服务网关

随着微服务的不断增多，不同的微服务一般会有不同的网络地址，而外部客户端可能需要调用多个服务的接口才能完成一个业务需求，如果让客户端直接与各个微服务通信可能出现：

1. 客户端需要调用不同的url地址，增加难度。
2. 在一定的场景下，存在跨域请求的问题。
3. 每个微服务都需要进行单独的身份认证。

​    为了解决这些问题，API网关顺势而生。

API网关直面意思是将所有API调用统一接入到API网关层，由网关层统一接入和输出。一个网关的基本功能有：统一接入、安全防护、协议适配、流量管控、长短链接支持、容错能力。有了网关之后，各个API服务提供团队可以专注于自己的的业务逻辑处理，而API网关更专注于安全、流量、路由等问题。

![image-20210506105213872](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/609bf030e00442778b0aef731aff641b~tplv-k3u1fbpfcp-watermark.awebp)

### 5.4、服务容错

在微服务当中，一个请求经常会涉及到调用几个服务，如果其中某个服务不可用，没有做服务容错的话，极有可能会造成一连串的服务不可用，这就是雪崩效应。

​    我们没法预防雪崩效应的发生，只能尽可能去做好容错。服务容错的三个核心思想是：

1. 不被外界环境影响。
2. 不被上游请求压垮。
3. 不被下游响应拖垮。

![image-20210506105412578](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcbac226daa24eb5a82c518b26104547~tplv-k3u1fbpfcp-watermark.awebp)

### 5.5、链路追踪

随着微服务架构的流行，服务按照不同的维度进行拆分，一次请求往往需要涉及到多个服务。互联网应用构建在不同的软件模块集上，这些软件模块，有可能是由不同的团队开发、可能使用不同的编程语言来实现、有可能布在了几千台服务器，横跨多个不同的数据中心。因此，就需要对一次请求涉及的多个服务链路进行日志记录，性能监控即链路追踪。

![image-20210506105549561](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4059e6664d1b4f9b9405bbbc995837de~tplv-k3u1fbpfcp-watermark.awebp)

## 6、微服务常见的解决方案

### 6.1、ServiceComb

![image-20210506105700397](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd847970ca5446cc954ef3ba83bc7ef8~tplv-k3u1fbpfcp-watermark.awebp)

Apache ServiceComb，前身是华为云的微服务引擎 CSE (Cloud Service Engine) 云服务，是全球首个Apache微服务顶级项目。它提供了一站式的微服务开源解决方案，致力于帮助企业、用户和开发者将企业应用轻松微服务化上云，并实现对微服务应用的高效运维管理。

### 6.2、SpringCloud

![image-20210506105744744](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a4e797ef25b46fcb6bad1b4a9b548ac~tplv-k3u1fbpfcp-watermark.awebp)

Spring Cloud是一系列框架的集合。它利用Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用Spring Boot的开发风格做到一键启动和部署。

Spring Cloud并没有重复制造轮子，它只是将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过Spring Boot风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。

### 6.3、Spring Cloud Alibaba

![image-20210506105837495](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73157a34247f43c9a7a090d258f36e44~tplv-k3u1fbpfcp-watermark.awebp)

Spring Cloud Alibaba 致力于提供微服务开发的一站式解决方案。此项目包含开发分布式应用微服务的必需组件，方便开发者通过 Spring Cloud 编程模型轻松使用这些组件来开发分布式应用服务。

 

 

 

 