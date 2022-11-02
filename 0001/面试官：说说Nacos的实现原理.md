面试官：说说Nacos的实现原理

## Nacos架构

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAMRLhicPTuWedpkkZ9cHCEg6fibICPMPTRIlTxCuloib4GSC4JVMdW4Q6g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- Provider APP：服务提供者
- Consumer APP：服务消费者
- Name Server：通过VIP（Virtual IP）或DNS的方式实现Nacos高可用集群的服务路由
- Nacos Server：Nacos服务提供者，里面包含的Open API是功能访问入口，Conig Service、Naming Service 是Nacos提供的配置服务、命名服务模块。Consitency Protocol是一致性协议，用来实现Nacos集群节点的数据同步，这里使用的是Raft算法（Etcd、Redis哨兵选举）
- Nacos Console：控制台

## 注册中心的原理

- 服务实例在启动时注册到服务注册表，并在关闭时注销
- 服务消费者查询服务注册表，获得可用实例
- 服务注册中心需要调用服务实例的健康检查API来验证它是否能够处理请求

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAF0cG1tLunlxAW4d7nSGe5KKZIkicfkCiahI6cUlmD7aHneTOqBRHAQvA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## SpringCloud完成注册的时机

在Spring-Cloud-Common包中有一个类`org.springframework.cloud. client.serviceregistry .ServiceRegistry` ,它是Spring Cloud提供的服务注册的标准。集成到Spring Cloud中实现服务注册的组件,都会实现该接口。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAiaibOBxzXJLj1lTgUHdy9VDqo326ibtwT7N5iaCtxI0zp3KvlmRbJSf4gw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

该接口有一个实现类是NacoServiceRegistry。

**SpringCloud集成Nacos的实现过程：**

在`spring-clou-commons`包的`META-INF/spring.factories`中包含自动装配的配置信息如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAS0xicrtftE3ttIhLxuWYKcVAGgdHdfTiayDyPprFJb1ZQpEiaLBaCRSIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其中`AutoServiceRegistrationAutoConfiguration`就是服务注册相关的配置类：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAwj2ZRyLyhUVENxvWDSSCwaz5jrvOhE3tDGHqHaWMEjia4aNIOibOicogA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在`AutoServiceRegistrationAutoConfiguration`配置类中,可以看到注入了一个`AutoServiceRegistration`实例,该类的关系图如下所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAjcBucMYXyT20LRiaGL3ea4M9sdnRoClGpHV5CmSGwdBNBpNlNlichEGw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看出, `AbstractAutoServiceRegistration`抽象类实现了该接口,并且最重要的是`NacosAutoServiceRegistration`继承了`AbstractAutoServiceRegistration`。

看到EventListener我们就应该知道，Nacos是通过Spring的事件机制继承到SpringCloud中去的。

`AbstractAutoServiceRegistration`实现了onApplicationEvent抽象方法,并且监听`WebServerInitializedEvent`事件(当Webserver初始化完成之后) , 调用`this.bind ( event )`方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAoenWVEwFAZaoNCdHPBxT9uj6c9XicckXH8t1yJibPqyWzZdRbN63OicFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最终会调用`NacosServiceREgistry.register()`方法进行服务注册。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAVd7kvK3IRwKEGCJL01Wxen0ZBrDiblOkgONByBbXKq65KtEO4jPtuGg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAOkTbth6uZdSy9ah4iaG7uj0YHlCpsJUlfvAj8Ox5AAF1WqXe6onuOqw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## NacosServiceRegistry的实现

在`NacosServiceRegistry.registry`方法中,调用了Nacos Client SDK中的`namingService.registerInstance`完成服务的注册。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSA5nN5aEbZU5HUeoiasqib4EaGMEfRy6vxIfjp6micTLAZLP55O23bj6DSA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

跟踪NacosNamingService的`registerInstance()`方法：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAeeQzDKQOyWKjP38xnEEKpyGibkHV3ukRNe3VhB30MAL80RJv7SiaNYFg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 通过`beatReactor.addBeatInfo()`创建心跳信息实现健康检测, Nacos Server必须要确保注册的服务实例是健康的,而心跳检测就是服务健康检测的手段。
- `serverProxy.registerService()`实现服务注册

**心跳机制：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSABLD1ic7R1by7Eiabic34gMjd2UuyDIuWWLsicG1M25pKXuGrnVXtVbaztg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上述代码看,所谓心跳机制就是客户端通过schedule定时向服务端发送一个数据包 ,然后启动-个线程不断检测服务端的回应,如果在设定时间内没有收到服务端的回应,则认为服务器出现了故障。Nacos服务端会根据客户端的心跳包不断更新服务的状态。

**注册原理：**

Nacos提供了SDK和Open API两种形式来实现服务注册。

Open API：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSApQo02LEb8yNXLPXmichicDyNRSLaNQPdKkJKxmebTtFzwb4A9OUibejVw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

SDK：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSA65zDIG5xXgZPMrfWUmbULjazoG6cJzBkZ9yOpTzpLhwpUzFmZ9YYicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这两种形式本质都一样，底层都是基于HTTP协议完成请求的。所以注册服务就是发送一个HTTP请求：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSADogeWVoDppOBgibJBGCn6paR0kdYDYCWB7DvjibGo5XcSaYU4CcHf1NA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对于nacos服务端，对外提供的服务接口请求地址为`nacos/v1/ns/instance`，实现代码咋`nacos-naming`模块下的InstanceController类中：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAABQs6icT4yQfJzp2EnteeV6kYooac3XW4kvF9drxPHSyAcXIRVBYM9w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 从请求参数汇总获得serviceName（服务名）和namespaceId（命名空间Id）
- 调用registerInstance注册实例

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAX8P1k1nOT3kmIyjeqbIAcXjGibNkI1O6cLibpfuicWqkocA4l76qGwclg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 创建一个控服务（在Nacos控制台“服务列表”中展示的服务信息），实际上是初始化一个serviceMap，它是一个ConcurrentHashMap集合
- getService，从serviceMap中根据namespaceId和serviceName得到一个服务对象
- 调用addInstance添加服务实例

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSARMBYqm4aSDXW1ufEoVOORITPSWlUvj5Abtl7iaM1iaf4BO5lE3AP6uwg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAQKFxaOfxrRQb1R7NQNf8PPRUjNibeHbHghP0sWNkjZL1kB9fZ0nYXQw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 根据namespaceId、serviceName从缓存中获取Service实例
- 如果Service实例为空，则创建并保存到缓存中

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSA8JyDOiaa1SicctVHu9LicqFYzsbkj7uEjMkLrcQduJwnVibtXWKZcVhjFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 通过`putService()`方法将服务缓存到内存
- `service.init()`建立心跳机制
- `consistencyService.listen`实现数据一致性监听

`service.init ( )`方法的如下图所示，它主要通过定时任务不断检测当前服务下所有实例最后发送心跳包的时间。如果超时,则设置healthy为false表示服务不健康,并且发送服务变更事件。

在这里请大家思考一一个问题,服务实例的最后心跳包更新时间是谁来触发的?实际上前面有讲到, Nacos客户端注册服务的同时也建立了心跳机制。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSA4S6e9yZe6t11K5p8N1LRanODhJEZu1dXr9YwicaOiawhwrAibC9abBzIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

putService方法，它的功能是将Service保存到serviceMap中：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAicnNBQh3zB5d8CYichlhHWJcNlaib5iaJxtVsSSduzo7PS5fPEf8Evbk5g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

继续调用addInstance方法把当前注册的服务实例保存到Service中：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAeQhWcbopBPeL3icFh5ibHial0Ujw4gkgrqL7oEoXtvibPWQj3mHaMVBglQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

总结：

- Nacos客户端通过Open API的形式发送服务注册请求

- Nacos服务端收到请求后，做以下三件事：

- - 构建一个Service对象保存到ConcurrentHashMap集合中
  - 使用定时任务对当前服务下的所有实例建立心跳检测机制
  - 基于数据一致性协议服务数据进行同步

## 服务提供者地址查询

Open API：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSA0SLXrqSFbs6Dz5b67hyNZe7C1uxDpTPzaNPCtOPGdbg62WJhn7TSlg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

SDK：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAFgM7IIl8mZKLzuUbhyVSp8aA9AaYZaLcmF3Xb175a93pQN0xvgCjeA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

InstanceController中的list方法：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAfiakt7YqBPJibfTNlWG3f4335rBciaaMHchfAWyCRtY6gYqCb54edXatQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 解析请求参数
- 通过doSrvIPXT返回服务列表数据

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSA1kvtgSBeSLzTThKZ1jNlejvIuKUic24afUZJNEY9DmiaMl5o20ycicaqQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAlMVpdVUOoJ58LV9NuH65BlSibkD6YX6NIXOe0picz3BoxTsF7xLj4liaw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 根据namespaceId、serviceName获得Service实例
- 从Service实例中基于srvIPs得到所有服务提供者实例
- 遍历组装JSON字符串并返回

## Nacos服务地址动态感知原理

可以通过subscribe方法来实现监听，其中serviceName表示服务名、EventListener表示监听到的事件：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAjiclgrKlwMkia0nlib38MTemSjrzZeh3LoeGLcaEwOsibDlwb0hhZvyCeQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

具体调用方式如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAM0urfkicWfnktNkorFBv2o3NvTibq11s6djcAXuxMdh8nNHDcB1YDqHQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

或者调用selectInstance方法，如果将subscribe属性设置为true，会自动注册监听：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAib9d1icuWH5wlHMAIRGYcU8vGLMvwAe7xC0cCf0oK05fWL8tUfYRTb6w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueTFNOceCjKWXWuBckIsVSAb10oYA28bE1icbnBpLjoheFHeArfPmcABbibBmw2xLwTsDXt8ULvChNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Nacos客户端中有一个HostReactor类，它的功能是实现服务的动态更新，基本原理是：

- 客户端发起时间订阅后，在HostReactor中有一个UpdateTask线程，每10s发送一次Pull请求，获得服务端最新的地址列表
- 对于服务端，它和服务提供者的实例之间维持了心跳检测，一旦服务提供者出现异常，则会发送一个Push消息给Nacos客户端，也就是服务端消费者
- 服务消费者收到请求之后，使用HostReactor中提供的processServiceJSON解析消息，并更新本地服务地址列表

萨达