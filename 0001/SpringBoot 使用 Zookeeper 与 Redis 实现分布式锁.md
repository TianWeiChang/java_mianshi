Spring Boot 使用 Zookeeper 与 Redis 实现分布式锁

环境准备 

- Redis 版本：5.0.7
- Java JDK 版本：1.8
- Redisson 版本：3.13.1
- SpringBoot 版本：2.3.4.RELEASE
- Zookeeper 版本：zookeeper-3.4.14

## 一、简介

### 1、什么是分布式锁

理解分布式锁之前，首先了解下什么是分布式架构，所谓 `分布式架构` 简单来说就是将相同或者不同服务部署在不同的服务器上，一个服务负责一个或多个功能，服务间通过网络协议进行通信。

那么，再来谈谈什么是分布式锁，所谓 `分布式锁` 就是 `控制分布式服务间同步访问共享资源的一种方式`，在分布式系统中，常常需要协调他们的动作。如果不同的服务或是同一个服务的不同实例之间共享了一个或一组资源，那么访问这些资源的时候，往往需要 `互斥` 来防止彼此干扰来 `保证一致性`，在这种情况下，便需要使用到分布式锁。

### 2、分布式锁使用场景

上面介绍了下什么是分布式锁，那么再说下它的使用场景：

比如，要开发商城系统时候使用 `分布式架构`，在分布式架构情况下的服务一般有多个实例。这里这个系统中的 `商品` 都会存在 `库存`，`库存服务` 也是分别有两个，分别为 `服务实例A`、`服务实例B` 实例，在同一时间内 `用户A` 和 `用户B` 同时对 `商品A` 进行下单，比如当前 `商品A` 的 `库存为 100`，然后执行 `两个下单命令` 后正常来说 `剩余库存应当为 98`, 不过，因为是分布式系统，两个用户执行下单后，下单指令被负载均衡到 `服务实例A`、`服务实例B`，两个实例同时对库存进行扣减操作，不过问题来了，它们在执行扣减前得先获取当前商品库存数量，于是它们都同时获取 `当前库存数 100`，然后 `服务实例A` 执行扣减库存 `100-1=99`，`服务实例B` 也执行扣减库存 `100-1=99`，最终 `商品A` 库存为 `99`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhrjWOK18cHRgQWuCd0Apd6xKL0ypFlXawsQRs6fHp0I67zyIPbXVazpcZfG8kYicnEg26uteDILd0A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

出现 `上面的情况`，其最主要的 `原因就是两个实例之前没有通信，无法得知在执行扣减库存时候，是否存另一个实例也正在操作更改库存数量`。所以，这时候如果有一个 `中间件` 可以存储这个公共的锁，这两个实例都通过这个 `中间件` 获取这把锁，谁能得到这把锁就能操作库存的更改，增加或者扣减库存，这样就能解决这个问题，这个就是分布式锁。

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhrjWOK18cHRgQWuCd0Apd6xiaJ4lTS6iaXswZ8S7qjx2xvQea4fP2FM1aicEHIe1tw3zJg83kad17s9g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3、分布式锁的实现方式

而分布式锁常用中间件有如下三种：

- 基于 Redis 实现;
- 基于 Zookeeper 实现;
- 基于数据库实现（性能差，问题多，所以不推荐，所以不过多叙述）;

### 4、分布式锁具备条件

- 具备可重入特性;
- 具备锁失效机制、防止死锁;
- 高可用、高性能的获取锁与释放锁;
- 具备非阻塞锁特性，即没有获取到锁直接返回获取锁失败;
- 在分布式系统环境下，一个方法在同一时间只能被一个机器的一个线程执行;

### 5、Redis 简介

**什么是 Redis：**

Redis 是一个高性能的 Key-Value 数据库，它是完全开源免费的，而且 Redis 是一个 NoSQL 类型数据库，是为了解决 高并发、高扩展，大数据存储 等一系列的问题而产生的数据库解决方案，是一个非关系型的数据库。

**使用 Redis 如何实现分布式锁：**

在 Java 的 Spring 框架中，常与 `spring-data-redis` 组件结合 `操作 Redis`，虽然里面有很多直接操作 Redis 的方法，不过遗憾的是其中并 `没有 Redis 锁` 的方法实现，其实现还是需要自己进行一些封装，且封装好的方法大多数都是针对 `Redis 单节点` 的，对于 `Redis 集群` 可能实现起来比较困难。

所以，常常需要使用 Redis 锁时，更推荐使用另一款组件 `Redisson`，该组件已经封装好了实现 `Redis 分布式锁` 的方法，我们可以轻松调用，且其也实现了 `红锁(支持 Redis 集群)`、`公平锁` 等等很多。Redisson 实现分布式锁的原理这里不过多叙述，推荐看 Redisson 实现分布式锁 这篇文章。

### 6、Zookeeper 简介

**什么是 Zookeeper：**

`ZooKeeper` 是一个分布式的，开放源码的分布式应用程序协调服务，是 Google 的 Chubby 一个开源的实现，是 Hadoop 和 Hbase 的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括 `配置维护`、`域名服务`、`分布式同步`、`组服务`等。

**Zookeeper 多层级节点：**

`Zookeeper` 提供一个多层级的节点命名空间 `znode`，每个节点都用一个以斜杠 `/` 分隔的路径表示，而且每个节点都有父节点（根节点除外），非常类似于文件系统。

例如 `/foo/doo` 这个表示一个 `znode`，它的父节点为 `/foo`，父父节点为 `/`，而 `/` 为根节点没有父节点。与文件系统不同的是，这些节点都可以设置关联的数据，而文件系统中只有文件节点可以存放数据而目录节点不行。`Zookeeper` 为了保证 `高吞吐` 和 `低延迟`，在内存中维护了这个树状的目录结构，这种特性使得 `Zookeeper`不能用于存放大量的数据，每个节点的存放数据上限为 1M。

为了保证高可用 `Zookeeper` 需要以 `集群` 形态来部署，这样只要集群中大部分机器是可用的（能够容忍一定的机器故障），那么 `Zookeeper` 本身仍然是可用的。客户端在使用 `Zookeeper` 时，需要知道集群机器列表，通过与集群中的某一台机器建立 TCP 连接来使用服务，客户端使用这个 TCP 链接来发送请求、获取结果、获取监听事件以及发送心跳包。如果这个连接异常断开了，客户端可以连接到另外的机器上。

**Zookeeper 的四种节点类型：**

- 持久节点（PERSISTENT）：创建节点的客户端与 Zookeeper 断开连接后，该节点依旧存在，这是默认的节点类型。
- 持久顺序节点（PERSISTENT_SEQUENTIAL）：创建节点时 Zookeeper 会根据创建的时间顺序给该节点名称进行编号。
- 临时节点（EPHEMERAL）：与持久节点相反，当创建节点的客户端与 Zookeeper 断开连接后，临时节点会被删除。
- 临时顺序节点（EPHEMERAL_SEQUENTIAL）：临时顺序节点结合和临时节点和顺序节点的特点，在创建节点时 Zookeeper 会根据创建的时间顺序给该节点名称进行编号，当创建节点的客户端与 Zookeeper 断开连接后，临时节点会被删除。

**Zookeeper 的事件监听：**

通过 Zookeeper 的事件监听机制可以让客户端收到节点状态变化。主要的事件类型有节点数据变化、节点的删除和创建。

**使用 Zookeeper 如何实现分布式锁：**

内容过多，这里不过多叙述，推荐看 10分钟看懂！ZooKeeper 典型应用场景：分布式锁 这篇文章。

## 二、使用 Spring 的 spring-data-redis 实现分布式锁

Spring 官方项目中存在 `spring-data-redis` 项目，内部封装了 `Lettuce` 客户端，所以它是一个非常利于操作 Redis 的组件，再结合 Redis 连接池能够非常方便的操作 Redis，这里我们使用该组件实现分布式锁。

> 需要注意的是，此方案不支持 Redis 集群模式

### 1、Maven 引入相关依赖

Maven 中引入需要的相关依赖，主要如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
    </parent>

    <groupId>mydlq.club</groupId>
    <artifactId>springboot-interface-idempotency</artifactId>
    <version>0.0.1</version>
    <name>springboot-interface-idempotency</name>
    <description>springboot interface idempotency</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <!--Lombok，业界常用基础组件，方便实现log方法与实体类get、set等-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!--SpringBoot-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--Spring-data-redis-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
         <!--为了使用 lettuce 线程池，必须使用该依赖-->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### 2、配置文件中设置相关参数

在 SpringBoot 的 `applicaiton.yaml` 配置文件中，设置连接 Redis 和连接池的参数，配置如下：

```
spring:
  redis:
    database: 0             #数据库
    #password: 123456       #数据库密码
    timeout: 1000           #超时时间
    host: 127.0.0.1         #Reids主机地址
    port: 6379              #Redis端口号
    lettuce:                #使用 lettuce 连接池
      pool:
        max-active: 20      #连接池最大连接数（使用负值表示没有限制）
        max-wait: -1        #连接池最大阻塞等待时间（使用负值表示没有限制）
        min-idle: 0         #连接池中的最大空闲连接
        max-idle: 10        #连接池中的最小空闲连接
```

### 3、创建分布式锁操作类

创建用于操作 Redis 实现分布式锁的方法，主要代码如下：

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.stereotype.Service;
import java.util.Arrays;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class RedisLock {

    /** RedisTemplate 对象 */
    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 尝试获取分布式锁
     *
     * @param key        分布式锁 key
     * @param value      分布式锁 value
     * @param expireTime 超时时间
     * @param timeUnit   超时时间单位
     * @return 是否成功获取分布式锁
     */
    public boolean tryLock(String key, String value, int expireTime, TimeUnit timeUnit) {
        Boolean result = redisTemplate.opsForValue().setIfAbsent(key, value, expireTime, timeUnit);
        if (Boolean.TRUE.equals(result)) {
            log.info("申请锁(" + key + "|" + value + ")成功");
            return true;
        }
        log.error("申请锁(" + key + "|" + value + ")失败");
        return false;
    }

    /**
     * 释放锁
     *
     * @param key   设置分布式锁 key
     * @param value 设置分布式锁 value
     */
    public void unLock(String key, String value) {
        // 设置 Lua 脚本，其中 KEYS[1] 是 key，KEYS[2] 是 value
        String script = "if redis.call('get', KEYS[1]) == KEYS[2] then return redis.call('del', KEYS[1]) else return 0 end";
        RedisScript<Long> redisScript = new DefaultRedisScript<>(script, Long.class);
        Long result = redisTemplate.execute(redisScript, Arrays.asList(key, value));
        if (result != null && result != 0L) {
            log.info("解锁(" + key + "|" + value + ")成功");
        } else {
            log.info("解锁(" + key + "|" + value + ")失败！");
        }
    }

}
```

> 这里的 unLock 方法中需要执行多个操作，所以这里使用 Lua 脚本保证执行的原子性。

### 4、创建 Controller 类并使用分布式锁

创建一个用于 `测试的 Controller 类` 来提供测试接口，里面[使用线程池](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247491717&idx=2&sn=c9a2ada120ee0a970a8ccc48ee283a65&chksm=fc798d2bcb0e043d22adf52e1e1325b2e14b9d5866e9777ece812832176b29bc7dd6da61ea1a&scene=21#wechat_redirect)来模拟多个线程调用，并且使用上面类中 `获取锁` 与 `解锁` 的方法。

```
import java.util.UUID;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * Redis Lock 测试接口
 *
 * @author mydlq
 */
@Slf4j
@RestController
public class LockController {

    /** Redis 分布式锁操作类*/
    @Autowired
    private RedisLock redisLock;

    /** 线程池*/
    ExecutorService executor = Executors.newFixedThreadPool(10);

    /**
     * 测试接口，这里循环创建多线程执行任务
     */
    @GetMapping("/lock")
    public void lockTest() {
        for (int i = 0; i < 1000; i++) {
            // 在线程池中，执行带分布式锁的业务逻辑
            executor.submit(() -> {
                executeLockService();
            });
        }
    }

    /**
     * 执行业务逻辑，并且使用分布式锁
     */
    private void executeLockService() {
        // 设置 Redis 键，该键名跟业务挂钩，应为一个不变的值，这里设置为 test
        String key = "test";
        // 生成 UUID，将该 UUID 最为 Redis 的值。
        // 注：设置个 UUID 随机值充当 Redis 存入的 Value 是为了保证，在分布式环境且存在多实例情况下，
        // 进行加锁和解锁操作的都是相同的进程（同一个实例），这样能够避免该锁被别的进程（实例）执行解锁操作。
        String value = UUID.randomUUID().toString();
        // 获取分布式锁，设置超时时间为 10 秒
        boolean execute = redisLock.tryLock(key, value, 10, TimeUnit.SECONDS);
        // 如果获取锁成功，则执行对应逻辑
        if (execute) {
            log.info("获取分布式锁，执行对应逻辑1");
            log.info("获取分布式锁，执行对应逻辑2");
            log.info("获取分布式锁，执行对应逻辑3");
            // 执行完成，释放分布式锁
            redisLock.unLock(key, value);
        }
    }

}
```

注意：上面设置 `UUID` 作为 `Redis 的值`，主要是是为了`保证加锁和解锁的进程都是同一个`，避免在分布式多实例下，”实例2”加的锁被”实例1”给解开这种情况。如果 `不设置` `value` 或者在 `多实例` 中都设置 `相同值`，那么可能发生下面情况，这里进行一下描述来加深理解。

一个应用存在多个实例，分别为实例1、实例2，然后执行过程中遇到如下情况：

- **1、** 实例1执行业务逻辑，获取分布式锁成功`（设置 key=test、value=不设置、过期时间=10 秒）`，然后正常执行接下来的业务逻辑代码。
- **2、** 实例1执行时间超过了 10 秒还没有执行完成，由于锁超过了指定超时时间，Redis 自动对其进行了删除（即释放了锁）。
- **3、** 实例2执行业务逻辑，也执行和实例1相同的方法，因为实例1已经释放了锁，所以锁获取成功`（key=test、value=不设置、过期时间=10 秒）`，然后也正常执行接下来的业务逻辑。
- **4、** 比如这时，实例1在又经过 2 秒时间后，终于执行完业务逻辑，然后执行释放锁`（key=test、value=不设置）`，这时候相当于将未完成业务逻辑的实例2的锁给释放掉了。
- **5、** 由于实例2的锁被实例1给释放掉了，所以如果这时候再进来新的请求，也调用该业务方法，那么也能顺利拿到锁执行业务逻辑，这样导致锁失去它的作用了。

所以，分布式锁一定要保证，哪个进程加的锁就该由哪个进行进行锁的释放，这里是分布式多实例情况，所以在执行加锁逻辑时，一定要设置个 UUID 唯一值来充当锁的值，在解锁时候也带上该值，来保证加锁和解锁的是在同一实例中，从而避免上述情况发生。

### 5、SpringBoot 启动类

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

## 三、使用 Redis 工具 Redisson 实现分布式锁

在上面例子中使用了 spring-data-redis 组件实现了分布式锁，不过在 `Java` 还有很多其它好用的 工具包能够直接操作 Redis，比如 `Jedis`、`Redisson` 等，这些工具对操作 Redis 进行了很多方法封装，非常好用，其中 Redis 官方比较推荐使用 `Redission` 这个工具，该工具对操作 Redis 进行很多封装，功能丰富，这里介绍下如何使用 `Redisson` 来实现 `分布式锁`。

这里说的 `Redisson` 是一个在 Redis 的基础上实现的 `Java` 驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的 `分布式` 的 `Java` 常用对象，还提供了许多分布式服务与使用 Redis 的最简单和最便捷的方法。`Redisson` 的宗旨是促进使用者对 Redis 的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

### 1、Maven 引入相关依赖

Maven 中引入需要的相关依赖，主要如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
    </parent>

    <groupId>mydlq.club</groupId>
    <artifactId>springboot-redisson-lock-example</artifactId>
    <version>0.0.1</version>
    <name>springboot-redisson-lock-example</name>
    <description>redisson distributed lock demo</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <!--Lombok，业界常用基础组件，方便实现log方法与实体类get、set等-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!--SpringBoot-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <!--Redisson-->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.13.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### 2、创建 Redisson 配置文件

在 `resources` 文件夹里面创建一个 `config` 文件夹，里面创建一个名为 `redisson-single.yaml` 的 **Redisson** 配置文件：

> 注意：下面配置中 `address: redis://IP:Port` 是必须配置的，还有就是如果 Redis 没有配置密码，则设置 `password` 参数为 `null`，不能为 `""`。

**redisson-single.yaml**

```
#单机模式
singleServerConfig:
  # 连接空闲超时，单位：毫秒
  idleConnectionTimeout: 10000
  # 连接超时，单位：毫秒
  connectTimeout: 10000
  # 命令等待超时，单位：毫秒
  timeout: 3000
  # 命令失败重试次数,如果尝试达到 retryAttempts（命令失败重试次数） 仍然不能将命令发送至某个指定的节点时，将抛出错误。
  # 如果尝试在此限制之内发送成功，则开始启用 timeout（命令等待超时） 计时。
  retryAttempts: 3
  # 命令重试发送时间间隔，单位：毫秒
  retryInterval: 1000
  # 密码
  password: null
  # 单个连接最大订阅数量
  subscriptionsPerConnection: 5
  # 客户端名称
  clientName: null
  # 节点地址
  address: redis://127.0.0.1:6379
  # 发布和订阅连接的最小空闲连接数
  subscriptionConnectionMinimumIdleSize: 1
  # 发布和订阅连接池大小
  subscriptionConnectionPoolSize: 50
  # 最小空闲连接数
  connectionMinimumIdleSize: 32
  # 连接池大小
  connectionPoolSize: 64
  # 数据库编号
  database: 1
  # DNS监测时间间隔，单位：毫秒
  dnsMonitoringInterval: 5000
# 线程池数量,默认值: 当前处理核数量 * 2
#threads: 0
# Netty线程池数量,默认值: 当前处理核数量 * 2
#nettyThreads: 0
# 编码
codec: !<org.redisson.codec.JsonJacksonCodec> {}
# 传输模式
transportMode : "NIO"
```

### 3、SpringBoot 配置文件引入 Redisson 配置文件

在 SpringBoot 的 `application` 配置文件中，设置 `redisson 配置文件` 的地址：

```
### 载入 Redisson 配置文件
spring:
  redis:
    redisson:
      config: classpath:config/redisson-single.yaml
```

### 4、创建 Controller 类并使用分布式锁

创建一个用于 `测试的 Controller 类` 来提供测试接口，里面使用线程池来模拟多个线程调用，并且使用 `Redisson` 锁方法实现 `分布式锁`。

```
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

@Slf4j
@RestController
public class LockController {

    /** Redisson 对象 */
    @Autowired
    private RedissonClient redissonClient;

    /** 线程池 */
    ExecutorService executor = Executors.newFixedThreadPool(10);

    @GetMapping("/lock")
    public void lockTest() {
        for (int i = 0; i < 1000; i++) {
            executor.submit(() -> {
                // 获取锁对象（可以为"可重入锁"、"公平锁",如果redis是集群模式，还可以使用"红锁"）
                //RLock lock = redissonClient.getFairLock("test");  //公平锁
                RLock lock = redissonClient.getLock("test");     //可重入锁
                try {
                    // 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
                    boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
                    // 如果获取锁成功，则执行对应逻辑
                    if (res) {
                        log.info("获取分布式锁，执行对应逻辑1");
                        log.info("获取分布式锁，执行对应逻辑2");
                        log.info("获取分布式锁，执行对应逻辑3");
                    }
                } catch (InterruptedException e) {
                    log.error("", e);
                    Thread.currentThread().interrupt();
                } finally {
                    lock.unlock();
                }
            });
        }
    }

}
```

### 5、SpringBoot 启动类

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

## 四、使用 Zookeeper 实现分布式锁

### 1、Maven 引入相关依赖

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
    </parent>

    <groupId>mydlq.club</groupId>
    <artifactId>springboot-zookeeper-lock-example</artifactId>
    <version>0.0.1</version>
    <name>springboot-zookeeper-lock-example</name>
    <description>zookeeper distributed lock demo</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <!-- SpringBoot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- Curator & Zookeeper -->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>5.1.0</version>
        </dependency>
        <!-- 创建 Zookeeper 客户端依赖，一定要和 Zookeeper Server 版本保持一致 -->
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.14</version>
            <exclusions>
                <!--因为 zk 包使用的是 log4j 日志，和 springboot 的logback 日志冲突 -->
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### 2、配置连接 Zookeeper 参数

在 SpringBoot 的 `applicaiton` 配置文件中，配置连接 `Zookeeper` 的参数，后面会配置一个读取配置的类来专门获取这些参数，配置如下：

```
zookeeper:
  address: 127.0.0.1:2181     #zookeeper Server地址,如果有多个,使用","隔离
                              #例如 ip1:port1,ip2:port2,ip3:port3
  retryCount: 5               #重试次数
  elapsedTimeMs: 5000         #重试间隔时间
  sessionTimeoutMs: 30000     #Session超时时间
  connectionTimeoutMs: 10000  #连接超时时间
```

### 3、创建读取配置文件中参数的类

从 `application` 配置文件中读取用于连接 `Zookeeper Server` 的参数：

```
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Data
@Configuration
@ConfigurationProperties(prefix = "zookeeper")
public class ZookeeperProperties {

    /** 重试次数 */
    private int retryCount;

    /** 重试间隔时间 */
    private int elapsedTimeMs;

    /**连接地址 */
    private String address;

    /**Session过期时间 */
    private int sessionTimeoutMs;

    /**连接超时时间 */
    private int connectionTimeoutMs;

}
```

### 4、创建连接 Zookeeper 的 Curator 客户端类

创建连接 `Zookeeper` 的 `Curator` 客户端的配置类，并设置初始化时进行连接：

```
import org.apache.curator.retry.RetryNTimes;
import org.apache.curator.framework.CuratorFramework;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.apache.curator.framework.CuratorFrameworkFactory;

@Configuration
public class ZookeeperConfig {

    /**
     * 创建 CuratorFramework 对象并连接 Zookeeper
     *
     * @param zookeeperProperties 从 Spring 容器载入 zookeeperProperties Bean 对象，读取连接 ZK 的参数
     * @return CuratorFramework
     */
    @Bean(initMethod = "start")
    public CuratorFramework curatorFramework(ZookeeperProperties zookeeperProperties) {
        return CuratorFrameworkFactory.newClient(
                zookeeperProperties.getAddress(),
                zookeeperProperties.getSessionTimeoutMs(),
                zookeeperProperties.getConnectionTimeoutMs(),
                new RetryNTimes(zookeeperProperties.getRetryCount(),
                        zookeeperProperties.getElapsedTimeMs()));
    }

}
```

### 5、创建分布式锁工具类

```
import lombok.extern.slf4j.Slf4j;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.util.concurrent.TimeUnit;

@Slf4j
@Component
public class LockUtil {

    @Autowired
    CuratorFramework curatorFramework;

    /**
     * 节点名称
     */
    public static final String NODE_PATH = "/lock-space/%s";

    /**
     * 尝试获取分布式锁
     *
     * @param key        分布式锁 key
     * @param expireTime 超时时间
     * @param timeUnit   时间单位
     * @return 超时时间单位
     */
    public InterProcessMutex tryLock(String key, int expireTime, TimeUnit timeUnit) {
        try {
            InterProcessMutex mutex = new InterProcessMutex(curatorFramework, String.format(NODE_PATH, key));
            boolean locked = mutex.acquire(expireTime, timeUnit);
            if (locked) {
                log.info("申请锁(" + key + ")成功");
                return mutex;
            }
        } catch (Exception e) {
            log.error("申请锁(" + key + ")失败,错误：{}", e);
        }
        log.warn("申请锁(" + key + ")失败");
        return null;
    }

    /**
     * 释放锁
     *
     * @param key          分布式锁 key
     * @param lockInstance InterProcessMutex 实例
     */
    public void unLock(String key, InterProcessMutex lockInstance) {
        try {
            lockInstance.release();
            log.info("解锁(" + key + ")成功");
        } catch (Exception e) {
            log.error("解锁(" + key + ")失败！");
        }
    }

}
```

### 6、创建 Controller 类并使用分布式锁

创建一个用于 `测试的 Controller 类` 来提供测试接口，里面[使用线程池](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247496923&idx=2&sn=2140cf92ccf00f4b136d83bd4665f56d&chksm=fc799975cb0e1063f5cab10cfc1ef7cfaa1d620c5f8185721bef2c931fd4c3191e4e8897c4c5&scene=21#wechat_redirect)来模拟多个线程调用，并且使用 `Curator` 锁方法实现 `分布式锁`。

```
import lombok.extern.slf4j.Slf4j;
import mydlq.club.lock.utils.LockUtil;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

@Slf4j
@RestController
public class LockController {

    /**
     * curatorFramework对象
     */
    @Autowired
    private LockUtil lockUtil;

    /**
     * 线程池
     */
    ExecutorService executor = Executors.newFixedThreadPool(10);

    @GetMapping("/lock")
    public void lockTest() {
        for (int i = 0; i < 1000; i++) {
            executor.submit(() -> {
                try {
                    String key = "test";
                    // 获取锁
                    InterProcessMutex lock = lockUtil.tryLock(key, 10, TimeUnit.SECONDS);
                    if (lock != null) {
                        // 如果获取锁成功，则执行对应逻辑
                        log.info("获取分布式锁，执行对应逻辑1");
                        log.info("获取分布式锁，执行对应逻辑2");
                        log.info("获取分布式锁，执行对应逻辑3");
                        // 释放锁
                        lockUtil.unLock(key, lock);
                    }
                } catch (Exception e) {
                    log.error("", e);
                }
            });
        }
    }

}
```

### 7、SpringBoot 启动类

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

## 五、其它

### 1、Redisson 连接 Redis 主从、哨兵、集群的配置文件

**主从配置文件 redisson-master-slave.yaml**

```
masterSlaveServersConfig:
  idleConnectionTimeout: 10000
  pingTimeout: 1000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  reconnectionTimeout: 3000
  failedAttempts: 3
  password: null
  subscriptionsPerConnection: 5
  clientName: null
  loadBalancer: !<org.redisson.connection.balancer.RoundRobinLoadBalancer> {}
  slaveSubscriptionConnectionMinimumIdleSize: 1
  slaveSubscriptionConnectionPoolSize: 50
  slaveConnectionMinimumIdleSize: 32
  slaveConnectionPoolSize: 64
  masterConnectionMinimumIdleSize: 32
  masterConnectionPoolSize: 64
  readMode: "SLAVE"
  slaveAddresses:
    - "redis://127.0.0.1:6381"
    - "redis://127.0.0.1:6380"
  masterAddress: "redis://127.0.0.1:6379"
  database: 0
threads: 0
nettyThreads: 0
codec: !<org.redisson.codec.JsonJacksonCodec> {}
"transportMode":"NIO"
```

**哨兵配置文件 redisson-sentinel.yaml**

```
sentinelServersConfig:
  idleConnectionTimeout: 10000
  pingTimeout: 1000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  reconnectionTimeout: 3000
  failedAttempts: 3
  password: null
  subscriptionsPerConnection: 5
  clientName: null
  loadBalancer: !<org.redisson.connection.balancer.RoundRobinLoadBalancer> {}
  slaveSubscriptionConnectionMinimumIdleSize: 1
  slaveSubscriptionConnectionPoolSize: 50
  slaveConnectionMinimumIdleSize: 32
  slaveConnectionPoolSize: 64
  masterConnectionMinimumIdleSize: 32
  masterConnectionPoolSize: 64
  readMode: "SLAVE"
  sentinelAddresses:
    - "redis://127.0.0.1:7001"
    - "redis://127.0.0.1:7002"
  masterName: "mymaster"
  database: 0
threads: 0
nettyThreads: 0
codec: !<org.redisson.codec.JsonJacksonCodec> {}
"transportMode":"NIO"
```

**集群配置文件 redisson-cluster.yaml**

```
clusterServersConfig:
  idleConnectionTimeout: 10000
  pingTimeout: 1000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  reconnectionTimeout: 3000
  failedAttempts: 3
  password: null
  subscriptionsPerConnection: 5
  clientName: null
  loadBalancer: !<org.redisson.connection.balancer.RoundRobinLoadBalancer> {}
  slaveSubscriptionConnectionMinimumIdleSize: 1
  slaveSubscriptionConnectionPoolSize: 50
  slaveConnectionMinimumIdleSize: 32
  slaveConnectionPoolSize: 64
  masterConnectionMinimumIdleSize: 32
  masterConnectionPoolSize: 64
  readMode: "SLAVE"
  nodeAddresses:
    - "redis://127.0.0.1:7001"
    - "redis://127.0.0.1:7002"
    - "redis://127.0.0.1:7003"
  scanInterval: 1000
threads: 0
nettyThreads: 0
codec: !<org.redisson.codec.JsonJacksonCodec> {}
"transportMode":"NIO"
```

## 示例项目地址：

Spring Boot 使用 Redis 实现分布式锁示例

> `https://github.com/my-dlq/blog-example/tree/master/springboot/springboot-redisson-lock-example`

Spring Boot 使用` Zookeeper `实现分布式锁示例

> `https://github.com/my-dlq/blog-example/tree/master/springboot/springboot-zookeeper-lock-example`

Spring Boot 使用 `spring-data-redis `实现分布式锁示例

> `https://github.com/my-dlq/blog-example/tree/master/springboot/springboot-data-redis-lock-example`

asdasdasd