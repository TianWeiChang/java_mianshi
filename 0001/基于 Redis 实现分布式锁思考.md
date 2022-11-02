## 基于 Redis 实现分布式锁(附代码)

Spring Cloud 分布式环境下，同一个服务都是部署在不同的机器上，这种情况无法像单体架构下数据一致性问题采用加锁就实现数据一致性问题，在高并发情况下，对于分布式架构显然是不合适的，针对这种情况我们就需要用到分布式锁了。

## 哪些场景需要用分布式锁

场景一:比较敏感的数据比如金额修改，同一时间只能有一个人操作，想象下2个人同时修改金额，一个加金额一个减金额，为了防止同时操作造成数据不一致，需要锁，如果是数据库需要的就是行锁或表锁，如果是在集群里，多个客户端同时修改一个共享的数据就需要分布式锁。

场景二:比如多台机器都可以定时执行某个任务，如果限制任务每次只能被一台机器执行，不能重复执行，就可以用分布式锁来做标记。

场景三:比如秒杀场景，要求并发量很高，那么同一件商品只能被一个用户抢到，那么就可以使用分布式锁实现。

## 分布式锁实现方式：

- 1、基于数据库实现分布式锁
- 2、基于缓存（redis，memcached，tair）实现分布式锁
- 3、基于Zookeeper实现分布式锁

为什么不使用数据库？

数据库是单点？搞两个数据库，数据之前双向同步。一旦挂掉快速切换到备库上。

没有失效时间？只要做一个定时任务，每隔一定时间把数据库中的超时数据清理一遍。

非阻塞的？搞一个while循环，直到insert成功再返回成功。

非重入的？在数据库表中加个字段，记录当前获得锁的机器的主机信息和线程信息，那么下次再获取锁的时候先查询数据库，如果当前机器的主机信息和线程信息在数据库可以查到的话，直接把锁分配给他就可以了。

大量请求下数据库往往是系统的瓶颈，大量连接，然后sql查询，几乎所有时间都浪费到这些上面，所以往往情况下能内存操作就在内存操作，使用基于内存操作的Redis实现分布式锁，也可以根据需求选择ZooKeeper 来实现。

通过 Redis 的 Redlock 和 ZooKeeper 来加锁，性能有了比较大的提升，一般情况我们根据实际场景选择使用。

## 分布式锁应该满足要求

- 互斥性 可以保证在分布式部署的应用集群中，同一个方法在同一时间只能被一台机器上的一个线程执行。
- 这把锁要是一把可重入锁（避免死锁）
- 不会发生死锁：有一个客户端在持有锁的过程中崩溃而没有解锁，也能保证其他客户端能够加锁
- 这把锁最好是一把阻塞锁（根据业务需求考虑要不要这条）
- 有高可用的获取锁和释放锁功能
- 获取锁和释放锁的性能要好

## Redis实现分布式锁

Redis实现分布式锁利用 `SETNX` 和 `SETEX`

基本命令主要有：

- SETNX(SET If Not Exists)：当且仅当 Key 不存在时，则可以设置，否则不做任何动作。

  当且仅当 key 不存在，将 key 的值设为 value ，并返回1；若给定的 key 已经存在，则 SETNX 不做任何动作，并返回0。

- SETEX：基于SETNX功能外,还可以设置超时时间，防止死锁。

### 分布式锁

分布式锁其实大白话，本质上要实现的目标(客户端)在redis中占一个位置，等到这个客户试用，别的人进来就必须得等着，等我试用完了，走了，你再来。 感觉跟多线程锁一样，意思大致是一样的，多线程是针对单机的，在同一个Jvm中，但是分布式石锁，是跨机器的，多个进程不同机器上发来得请求，去对同一个数据进行操作。

比如，分布式架构下的秒杀系统，几万人对10个商品进行抢购，10个商品存在redis中，就是表示10个位置，第一个人进来了，商品就剩9个了，第二个人进来就剩8个，在第一个人进来的时候，其他人必须等到10个商品数量成功减去1之后你才能进来。

这个过程中第一个人进来的时候还没操作减1然后异常了，没有释放锁，然后后面人一直等待着，这就是死锁。真对这种情况可以设置超时时间，如果超过10s中还是没出来，就让他超时失效。

redis中提供了 `setnx(set if not exists)` 指令

```
> setnx lock:codehole true  -- 锁定
OK
... do something xxxx...  数量减1
> del lock:codehole           -- 释放锁
(integer) 1  --成功
```

如果在减1期间发生异常 del 指令没有被调用 然后就一直等着，锁永远不会释放。

redis Redis 2.8 版本中提供了 setex(set if not exists) 指令 setnx 和 expire 两个指令构成一个原子操作 给锁加上一个过期时间

```
> setex lock:codehole true
OK
> expire lock:codehole 5
... do something xxxx ...
> del lock:codehole
(integer) 1
```

`SETEX 实现原理`

通过 SETNX 设置 Key-Value 来获得锁，随即进入死循环，每次循环判断，如果存在 Key 则继续循环，如果不存在 Key，则跳出循环，当前任务执行完成后，删除 Key 以释放锁。

### 实现步骤

pom.xml 导入Redis依赖

```xml
<!-- redis-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.10</version>
    <scope>provided</scope>
</dependency>
```

### 添加配置文件 application.yml：

```
server:
  port: 8080

spring:
  profiles: dev
  data:
  redis:
    # Redis数据库索引（默认为0）
    database: 0
    # Redis服务器地址
    host: 127.0.0.1
    # Redis服务器连接端口
    port: 6379
    # Redis服务器连接密码（默认为空）
    password:
```

### 全局锁类

```
@Data
public class Lock {
    /**
     * key名
     */
    private String name;
    /**
     * value值
     */
    private String value;

    public Lock(String name, String value) {
        this.name = name;
        this.value = value;
    }

}
```

### 分布式锁类

```
@Slf4j
@Component
public class DistributedLockConfig {

    /**
     * 单个业务持有锁的时间30s，防止死锁
     */
    private final static long LOCK_EXPIRE = 30 * 1000L;
    /**
     * 默认30ms尝试一次
     */
    private final static long LOCK_TRY_INTERVAL = 30L;
    /**
     * 默认尝试20s
     */
    private final static long LOCK_TRY_TIMEOUT = 20 * 1000L;

    private RedisTemplate template;

    public void setTemplate(RedisTemplate template) {
        this.template = template;
    }

    /**
     * 尝试获取全局锁
     *
     * @param lock 锁的名称
     * @return true 获取成功，false获取失败
     */
    public boolean tryLock(Lock lock) {
        return getLock(lock, LOCK_TRY_TIMEOUT, LOCK_TRY_INTERVAL, LOCK_EXPIRE);
    }

    /**
     * 尝试获取全局锁
     * SETEX：可以设置超时时间
     *
     * @param lock    锁的名称
     * @param timeout 获取超时时间 单位ms
     * @return true 获取成功，false获取失败
     */
    public boolean tryLock(Lock lock, long timeout) {
        return getLock(lock, timeout, LOCK_TRY_INTERVAL, LOCK_EXPIRE);
    }

    /**
     * 尝试获取全局锁
     *
     * @param lock        锁的名称
     * @param timeout     获取锁的超时时间
     * @param tryInterval 多少毫秒尝试获取一次
     * @return true 获取成功，false获取失败
     */
    public boolean tryLock(Lock lock, long timeout, long tryInterval) {
        return getLock(lock, timeout, tryInterval, LOCK_EXPIRE);
    }

    /**
     * 尝试获取全局锁
     *
     * @param lock           锁的名称
     * @param timeout        获取锁的超时时间
     * @param tryInterval    多少毫秒尝试获取一次
     * @param lockExpireTime 锁的过期
     * @return true 获取成功，false获取失败
     */
    public boolean tryLock(Lock lock, long timeout, long tryInterval, long lockExpireTime) {
        return getLock(lock, timeout, tryInterval, lockExpireTime);
    }

    /**
     * 操作redis获取全局锁
     *
     * @param lock           锁的名称
     * @param timeout        获取的超时时间
     * @param tryInterval    多少ms尝试一次
     * @param lockExpireTime 获取成功后锁的过期时间
     * @return true 获取成功，false获取失败
     */
    public boolean getLock(Lock lock, long timeout, long tryInterval, long lockExpireTime) {

        try {
            if (StringUtils.isEmpty(lock.getName()) || StringUtils.isEmpty(lock.getValue())) {
                return false;
            }
            long startTime = System.currentTimeMillis();
            do {
                if (!template.hasKey(lock.getName())) {
                    ValueOperations<String, String> ops = template.opsForValue();
                    ops.set(lock.getName(), lock.getValue(), lockExpireTime, TimeUnit.MILLISECONDS);
                    return true;
                } else {
                    //存在锁
                    log.debug("lock is exist!！！");
                }

                //尝试超过了设定值之后直接跳出循环
                if (System.currentTimeMillis() - startTime > timeout) {
                    return false;
                }

                //每隔多长时间尝试获取
                Thread.sleep(tryInterval);
            }
            while (template.hasKey(lock.getName()));
        } catch (InterruptedException e) {
            log.error(e.getMessage());
            return false;
        }
        return false;
    }

    /**
     * 获取锁
     * SETNX(SET If Not Exists)：当且仅当 Key 不存在时，则可以设置，否则不做任何动作。
     */
    public Boolean getLockNoTime(Lock lock) {
        if (!StringUtils.isEmpty(lock.getName())) {
            return false;
        }

        // setIfAbsent 底层封装命令 是 setNX()
        boolean falg = template.opsForValue().setIfAbsent(lock.getName(), lock.getValue());

        return false;
    }

    /**
     * 释放锁
     */
    public void releaseLock(Lock lock) {
        if (!StringUtils.isEmpty(lock.getName())) {
            template.delete(lock.getName());
        }
    }

}
```

### 测试方法

```
@RequestMapping("test")
    public String index() {
        distributedLockConfig.setTemplate(redisTemplate);
        Lock lock = new Lock("test", "test");
        if (distributedLockConfig.tryLock(lock)) {
            try {
                //为了演示锁的效果，这里睡眠5000毫秒
                System.out.println("执行方法");
                Thread.sleep(5000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            distributedLockConfig.releaseLock(lock);
        }
        return "hello world!";
    }
```

开启两个浏览器窗口，执行方法，我们可以看到两个浏览器在等待执行，当一个返回 hello world! 之后，如果没超时执行另一个也会返回hello world! 两个方法彼此先后返回，说明分布式锁执行成功。

但是存在一个问题：

这段方法是先去查询key是否存在redis中，如果存在走循环，然后根据间隔时间去等待尝试获取，如果不存在则进行获取锁，如果等待时间超过超时时间返回false。

- 1 这种方式性能问题很差，每次获取锁都要进行等待，很是浪费资源，
- 2 如果在判断锁是否存在这儿2个或者2个以上的线程都查到redis中存在key，同一时刻就无法保证一个客户端持有锁，不具有排他性。

如果在集群环境下也会存在问题

假如在哨兵模式中 主节点获取到锁之后,数据没有同步到从节点主节点挂掉了，这样数据完整性不能保证，另一个客户端请求过来，就会一把锁被两个客户端持有，会导致数据一致性出问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjGvwtgsicYazGjlicSg3HtmiceLhAqPrrMUMptl3aMG8U6MRichibXOVOXHfZmADLQ3g7NqAWwr0lviaQibA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述

对此Redis中还提供了另外一种实现分布式锁的方法 `Redlock`

## 利用 Redlock

Redlock是redis官方提出的实现分布式锁管理器的算法。这个算法会比一般的普通方法更加安全可靠。

为什么选择红锁？ 在集群中需要半数以上的节点同意才能获得锁，保证了数据的完整性，不会因为主节点数据存在，主节点挂了之后没有同步到从节点，导致数据丢失。

Redlock 算法

使用场景

对于Redis集群模式尽量采用这种分布式锁，保证高可用，数据一致性，就使用Redlock 分布式锁。

pom.xml 增加依赖

```
<dependency>
    <groupId>org.redisson</groupId>
        <artifactId>redisson</artifactId>
        <version>3.7.0</version>
</dependency>
```

获取锁后需要处理的逻辑

```
/**
 * 获取锁后需要处理的逻辑
 */
public interface AquiredLockWorker<T> {
    T invokeAfterLockAquire() throws Exception;
}
```

获取锁管理类

```
/**
 * 获取锁管理类
 */
public interface DistributedLocker {

    /**
     * 获取锁
     * @param resourceName  锁的名称
     * @param worker 获取锁后的处理类
     * @param <T>
     * @return 处理完具体的业务逻辑要返回的数据
     * @throws UnableToAquireLockException
     * @throws Exception
     */
    <T> T lock(String resourceName, AquiredLockWorker<T> worker) throws UnableToAquireLockException, Exception;

    <T> T lock(String resourceName, AquiredLockWorker<T> worker, int lockTime) throws UnableToAquireLockException, Exception;

}
```

异常类

```java
/**
 * 异常类
 */
public class UnableToAquireLockException extends RuntimeException {

    public UnableToAquireLockException() {
    }

    public UnableToAquireLockException(String message) {
        super(message);
    }

    public UnableToAquireLockException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

获取RedissonClient连接类

```java
/**
 * 获取RedissonClient连接类
 */
@Component
public class RedissonConnector {
    RedissonClient redisson;
    @PostConstruct
    public void init(){
        redisson = Redisson.create();
    }

    public RedissonClient getClient(){
        return redisson;
    }

}
```

分布式锁实现

```java
@Component
public class RedisLocker  implements DistributedLocker{

    private final static String LOCKER_PREFIX = "lock:";

    @Autowired
    RedissonConnector redissonConnector;
    @Override
    public <T> T lock(String resourceName, AquiredLockWorker<T> worker) throws InterruptedException, UnableToAquireLockException, Exception {

        return lock(resourceName, worker, 100);
    }

    @Override
    public <T> T lock(String resourceName, AquiredLockWorker<T> worker, int lockTime) throws UnableToAquireLockException, Exception {
        RedissonClient redisson= redissonConnector.getClient();
        RLock lock = redisson.getLock(LOCKER_PREFIX + resourceName);
        // Wait for 100 seconds seconds and automatically unlock it after lockTime seconds
        boolean success = lock.tryLock(100, lockTime, TimeUnit.SECONDS);
        if (success) {
            try {
                return worker.invokeAfterLockAquire();
            } finally {
                lock.unlock();
            }
        }
        throw new UnableToAquireLockException();
    }
}
```

测试方法

```java
ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(10);
        for (int i = 0; i < 50; i++) {
            scheduledExecutorService.execute(new Worker());
        }
        scheduledExecutorService.shutdown();

 //任务
class Worker implements Runnable {
        public Worker() {
        }
        @Override
        public void run() {
            try {
                redisLocker.lock("tizz1100", new AquiredLockWorker<Object>() {
                    @Override
                    public Object invokeAfterLockAquire() {
                        doTask();
                        return null;
                    }
                });
            } catch (Exception e) {
            }
        }

        void doTask() {
            System.out.println(Thread.currentThread().getName() + " ---------- " + LocalDateTime.now());
            System.out.println(Thread.currentThread().getName() + " start");
            Random random = new Random();
            int _int = random.nextInt(200);
            System.out.println(Thread.currentThread().getName() + " sleep " + _int + "millis");
            try {
                Thread.sleep(_int);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " end");
        }
}
```