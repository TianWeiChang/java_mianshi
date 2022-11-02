Redis使用不当，把应用搞挂了！

## 前言

>  今年以来，运气都不太好。公司在大力发展，招了不少新同事。新来的同事在使用 Redis 时，写了一个 Bug，导致应用卡死。 

老板直接批评了我，说我也有连带责任，怎么带的团队，质量不过关，造成重量级生产事故，好在未造成财产损失！

首先说下问题现象：内网 sandbox 环境 API 持续 1 周出现应用卡死，所有 API 无响应现象。

刚开始当测试抱怨环境响应慢的时候 ，我们重启一下应用应用恢复正常，于是没做处理。

但是后来问题出现频率越来越频繁，越来越多的同事开始抱怨，于是感觉代码可能有问题开始排查。

## 问题排查

首先发现，开发的本地 IDE 没有发现问题。应用卡死时候数据库，Redis 都正常，并且无特殊错误日志。开始怀疑是 sandbox 环境机器问题（测试环境本身就很脆弱）。

于是 ssh 登陆服务器 ，执行以下命令：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XQjOsOibWOHNe5nAicWjL6QUJ7BP5SbDzybMe7wyGh6KcPpVSKxg9kRdQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这时发现机器还算正常，但是内心还是😖，于是打算看下 JVM 堆栈信息。先看下问题应用比较耗资源的线程。

执行 top -H -p 12798，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XWs2akd7xtxEhGYP6hbEeILqtL13h8wLnA7L73WCVaLwq3Y08CTur4Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

找到前 3 个相对比较耗资源的线程：

- jstack 查看堆内存。
- jstack 12798 |grep 12799 的 16 进制 31ff。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XNo32XYep6a8K8Vo1dCJOVcOwIVLgXmLInBjkxW4E5quPhvA9QQ1Ekw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

没看出什么问题，上下 10 行也看看。于是执行：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XXYsl3Dx3d6U3RO1rTrmhoIwib5XDBnoVpek2wBOuQ1LImI6o8QNKXicg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

看到一些线程都是处于 lock 状态。但没有出现业务相关的代码，忽略了。这时候没有什么头绪。思考一番。决定放弃这次卡死状态的机器。

为了保护事故现场，先 dump 了问题进程所有堆内存，然后 debug 模式重启测试环境应用，打算问题再显时直接远程 debug 问题机器。

## 问题再现

第二天问题再现，于是通知运维 Nginx 转发拿掉这台问题应用，自己远程 debug Tomcat。

自己随意找了一个接口，断点在接口入口地方。悲剧开始，什么也没有发生！API 等待服务响应，没进断点。

这时候有点懵逼，冷静了一会，在入口之前的 AOP 地方下了个断点，再 debug 一次，这次进了断点。F8 N 次后发现在执行 Redis 命令的时候卡住了。

继续跟踪，最后在到 jedis 的一个地方发现问题：

```
/**
 * Returns a Jedis instance to be used as a Redis connection. The instance can be newly created or retrieved from a
 * pool.
 * 
 * @return Jedis instance ready for wrapping into a {@link RedisConnection}.
 */
protected Jedis fetchJedisConnector() {
   try {
      if (usePool && pool != null) {
         return pool.getResource();
      }
      Jedis jedis = new Jedis(getShardInfo());
      // force initialization (see Jedis issue #82)
      jedis.connect();
      return jedis;
   } catch (Exception ex) {
      throw new RedisConnectionFailureException("Cannot get Jedis connection", ex);
   }
}
```

上面 pool.getResource() 后线程开始 wait。

```
public T getResource() {
  try {
    return internalPool.borrowObject();
  } catch (Exception e) {
    throw new JedisConnectionException("Could not get a resource from the pool", e);
  }
}
```

return internalPool.borrowObject()；这个代码应该是一个租赁的代码。接着跟踪：

```
public T borrowObject(long borrowMaxWaitMillis) throws Exception {
    this.assertOpen();
    AbandonedConfig ac = this.abandonedConfig;
    if (ac != null && ac.getRemoveAbandonedOnBorrow() && this.getNumIdle() < 2 && this.getNumActive() > this.getMaxTotal() - 3) {
        this.removeAbandoned(ac);
    }

    PooledObject<T> p = null;
    boolean blockWhenExhausted = this.getBlockWhenExhausted();
    long waitTime = 0L;

    while(p == null) {
        boolean create = false;
        if (blockWhenExhausted) {
            p = (PooledObject)this.idleObjects.pollFirst();
            if (p == null) {
                create = true;
                p = this.create();
            }

            if (p == null) {
                if (borrowMaxWaitMillis < 0L) {
                    p = (PooledObject)this.idleObjects.takeFirst();
                } else {
                    waitTime = System.currentTimeMillis();
                    p = (PooledObject)this.idleObjects.pollFirst(borrowMaxWaitMillis, TimeUnit.MILLISECONDS);
                    waitTime = System.currentTimeMillis() - waitTime;
                }
            }

            if (p == null) {
                throw new NoSuchElementException("Timeout waiting for idle object");
            }
```

其中有段代码：

```
if (p == null) {
    if (borrowMaxWaitMillis < 0L) {
        p = (PooledObject)this.idleObjects.takeFirst();
    } else {
        waitTime = System.currentTimeMillis();
        p = (PooledObject)this.idleObjects.pollFirst(borrowMaxWaitMillis, TimeUnit.MILLISECONDS);
        waitTime = System.currentTimeMillis() - waitTime;
    }
}
```

borrowMaxWaitMillis<0 会一直执行，然后一直循环了。开始怀疑这个值没有配置。

找到 Redis pool 配置，发现确实没有配置 MaxWaitMillis，配置后 else 代码也是一个 Exception 并不能解决问题。

继续 F8：

```
public E takeFirst() throws InterruptedException {
    this.lock.lock();

    Object var2;
    try {
        Object x;
        while((x = this.unlinkFirst()) == null) {
            this.notEmpty.await();
        }

        var2 = x;
    } finally {
        this.lock.unlock();
    }

    return var2;
}
```

到这边发现 lock 字眼，开始怀疑所有请求 API 都被阻塞了。

于是再次 ssh 服务器安装 arthas（Arthas 是 Alibaba 开源的 Java 诊断工具）执行 thread 命令：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XCzbs4BurmNpbcfJGmwaKGL99KYrxVDICM9ywupAWQHFhicaRJib7IicUA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

发现大量 http-nio 的线程 waiting 状态，http-nio-8083-exec- 这个线程其实就是出来 HTTP 请求的 Tomcat 线程。

随意找一个线程查看堆内存 thread -428：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XvFSJ0d2pkoOgias6JYaic4dojVCNmRnQqrQS8RLzGW8AuPzwJ24uwmEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这是能确认就是 API 一直转圈的问题，就是这个 Redis 获取连接的代码导致的。

解读这段内存代码，所有线程都在等 `@53e5504e` 这个对象释放锁。于是` jstack `全局搜了` 53e5504e `，没有找到这个对象所在线程。

自此，问题原因能确定是 Redis 连接获取的问题。但是什么原因造成获取不到连接的还不能确定。

再次执行 arthas 的 `thread -b`（thread -b 找出当前阻塞其他线程的线程）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XiaB7scqYwNQq70XhFh1sWsLnnJv0BN4XnIDH9Y4DMcpc9eVn1uQcR2g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

没有结果。这边和想的不一样，应该是能找到一个阻塞线程的，于是看了下这个命令的文档，发现有下面这句话：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7Xt5nRBO8cO4ictHzG0UPLKJQ95V1uqcvibeNia4uMe0NlVOonjPia5XY3ibg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

好吧，我们刚好是后者... 再次整理思路。

这次修改 Redis pool 配置，将获取连接超时时间设置为 2s，然后等问题再次复现时观察应用最后正常时干过什么。

添加以下配置：

```
JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory();
.......
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxWaitMillis(2000);
.......
jedisConnectionFactory.afterPropertiesSet();
```

重启服务，等待... 又过一天，再次复现。

ssh 登陆服务器，检查 Tomcat accesslog，发现大量 API 请求出现 500。

```
org.springframework.data.redis.RedisConnectionFailureException: Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource fr
om the pool
    at org.springframework.data.redis.connection.jedis.JedisConnectionFactory.fetchJedisConnector(JedisConnectionFactory.java:140)
    at org.springframework.data.redis.connection.jedis.JedisConnectionFactory.getConnection(JedisConnectionFactory.java:229)
    at org.springframework.data.redis.connection.jedis.JedisConnectionFactory.getConnection(JedisConnectionFactory.java:57)
    at org.springframework.data.redis.core.RedisConnectionUtils.doGetConnection(RedisConnectionUtils.java:128)
    at org.springframework.data.redis.core.RedisConnectionUtils.getConnection(RedisConnectionUtils.java:91)
    at org.springframework.data.redis.core.RedisConnectionUtils.getConnection(RedisConnectionUtils.java:78)
    at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:177)
    at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:152)
    at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:85)
    at org.springframework.data.redis.core.DefaultHashOperations.get(DefaultHashOperations.java:48)
```

找到源头第一次出现 500 地方，发现以下代码：

```
.......
Cursor c = stringRedisTemplate.getConnectionFactory().getConnection().scan(options);
while (c.hasNext()) {
.....,,
   }
```

分析这段代码：

```
stringRedisTemplate.getConnectionFactory().getConnection()
```

获取 pool 中的 redisConnection 后，并没有后续操作。

也就是说此时 Redis 连接池中的链接被租赁后，并没有释放或者退还到链接池中、虽然业务已处理完毕 redisConnection 已经空闲，但是 pool 中的 redisConnection 的状态还没有回到 idle 状态。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7X4h7ibmISd3tfwPOKjK3lMCA9wEX8Z2nFfqdVD9bB3pIygIKxKJX4spA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

正常应为：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eZzl4LXykQxezMu5ndlMl6q8qjvvZh7XTxWzDEZOshMQPCoACkOVkXgiam7eNTDIFEq5X1EbQFURc0TnN1ly8icw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

自此问题已经找到。

## 总结

Spring `stringRedisTemplate `对` Redis `常规操作做了一些封装，但还不支持像 `Scan SetNx `等命令。这时需要拿到 `jedis Connection `进行一些特殊的` Commands`。

不推荐使用：

```
stringRedisTemplate.getConnectionFactory().getConnection()
```

我们可以使用下面代码来执行：

```java
stringRedisTemplate.execute(new RedisCallback<Cursor>() {

     @Override
     public Cursor doInRedis(RedisConnection connection) throws DataAccessException {

       return connection.scan(options);
     }
   });
```

或者使用完 connection 后执行：

```java
RedisConnectionUtils.releaseConnection(conn, factory);
```

来释放 Connection。

同时，Redis 中也不建议使用 keys 命令。Redis Pool 的配置应该合理设置，否则出现问题无错误日志无报错，定位相当困难。

