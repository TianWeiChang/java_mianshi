面试官：Java中对象池的本质是什么？

## 简介

对象池顾名思义就是存放对象的池，与我们常听到的线程池、数据库连接池、http连接池等一样，都是典型的池化设计思想。

对象池的优点就是可以集中管理池中对象，减少频繁创建和销毁长期使用的对象，从而提升复用性，以节约资源的消耗，可以有效避免频繁为对象分配内存和释放堆中内存，进而减轻jvm垃圾收集器的负担，避免内存抖动。

`Apache Common Pool2` 是Apache提供的一个通用对象池技术实现，可以方便定制化自己需要的对象池，大名鼎鼎的 `Redis` 客户端` Jedis` 内部连接池就是基于它来实现的。

## 核心接口

`Apache Common Pool2 `的核心内部类如下：

- `ObjectPool`：对象池接口，对象池实体，取用对象的地方

- - 对象的提供与归还(工厂来操作)：`borrowObject` `returnObject`
  - 创建对象(使用工厂来创建)：`addObject`
  - 销毁对象(使用工厂来销毁)：`invalidateObject`
  - 池中空闲对象数量、被使用对象数量：`getNumActive` `getNumIdle`

- `PooledObject`：被包装的对象，是池中的对象，除了对象本身之外包含了创建时间、上次被调用时间等众多信息

- `PooledObjectFactory`：对象工厂，管理对象的生命周期，提供了对象创建、销毁、验证、钝化、激活等一系列功能

- `BaseObjectPoolConfig`：提供一些必要的配置，例如空闲队列是否先进先出、工厂创建对象前是否需要测试、对象从对象池取出时是否测试等基础属性，`GenericObjectPoolConfig`继承了本类做了默认配置，我们在实际使用中继承它即可，可以结合业务情况扩展对象池配置，例如数据库连接池线程前缀、字符串池长度或名称规则等

- `KeyedObjectPool<K,V>`：键值对形式的对象池接口，使用场景很少

- `KeyedPooledObjectFactory<K,V>`：同上，为键值对对象池管理对象的工厂

## 池对象的状态

查看源码`PooledObjectState`枚举下列出了池对象所有可能处于的状态。

```java
public enum PooledObjectState {
    //在空闲队列中,还未被使用
    IDLE,
    //使用中
    ALLOCATED,
    //在空闲队列中,当前正在测试是否满足被驱逐的条件
    EVICTION,
   //不在空闲队列中，目前正在测试是否可能被驱逐。因为在测试过程中，试图借用对象，并将其从队列中删除。
    //回收测试完成后，它应该被返回到队列的头部。
    EVICTION_RETURN_TO_HEAD,
   //在队列中，正在被校验
    VALIDATION,
   //不在队列中，当前正在验证。该对象在验证时被借用，由于配置了testOnBorrow，
    //所以将其从队列中删除并预先分配。一旦验证完成，就应该分配它。
    VALIDATION_PREALLOCATED,
   //不在队列中，当前正在验证。在之前测试是否将该对象从队列中移除时，曾尝试借用该对象。
    //一旦验证完成，它应该被返回到队列的头部。
    VALIDATION_RETURN_TO_HEAD,
   //无效状态(如驱逐测试或验证)，并将/已被销毁
    INVALID,
   //判定为无效,将会被设置为废弃
    ABANDONED,
   //正在使用完毕,返回池中
    RETURNING
}
```

状态理解

- `abandoned `：被借出后，长时间未被使用则被标记为该状态。如代码所示，当该对象处于`ALLOCATED`状态，即被借出使用中，距离上次被使用的时间超过了设置的`getRemoveAbandonedTimeout`则被标记为废弃。

```java
private void removeAbandoned(final AbandonedConfig abandonedConfig) {
    // Generate a list of abandoned objects to remove
    final long now = System.currentTimeMillis();
    final long timeout =
            now - (abandonedConfig.getRemoveAbandonedTimeout() * 1000L);
    final ArrayList<PooledObject<T>> remove = new ArrayList<>();
    final Iterator<PooledObject<T>> it = allObjects.values().iterator();
    while (it.hasNext()) {
        final PooledObject<T> pooledObject = it.next();
        synchronized (pooledObject) {
            if (pooledObject.getState() == PooledObjectState.ALLOCATED &&
                    pooledObject.getLastUsedTime() <= timeout) {
                pooledObject.markAbandoned();
                remove.add(pooledObject);
            }
        }
    }
```

## 流程理解

1.对象真实是存储在哪里？

```java
 private PooledObject<T> create() throws Exception {
    .....
   final PooledObject<T> p;
        try {
            p = factory.makeObject();
        .....
        allObjects.put(new IdentityWrapper<>(p.getObject()), p);
        return p;
 }
```

我们查看`allObjects`，所有对象都存储于`ConcurrentHashMap`，除了被杀掉的对象。

```
/*
 * All of the objects currently associated with this pool in any state. It
 * excludes objects that have been destroyed. The size of
 * {@link #allObjects} will always be less than or equal to {@link
 * #_maxActive}. Map keys are pooled objects, values are the PooledObject
 * wrappers used internally by the pool.
 */
private final Map<IdentityWrapper<T>, PooledObject<T>> allObjects =
    new ConcurrentHashMap<>();
```

2.取用对象的逻辑归纳如下

- 首先根据`AbandonedConfig`配置判断是否取用对象前执行清理操作

- 再从`idleObject`中尝试获取对象，获取不到就创建新的对象

- - 判断`blockWhenExhausted`是否设置为`true`，(这个配置的意思是当对象池的`active`状态的对象数量已经达到最大值`maxinum`时是否进行阻塞直到有空闲对象)
  - 是的话按照设置的`borrowMaxWaitMillis`属性等待可用对象

- 有可用对象后调用工厂的`factory.activateObject`方法激活对象

- 当`getTestOnBorrow`设置为`true`时，调用`factory.validateObject(p)`对对象进行校验，通过校验后执行下一步

- 调用`updateStatsBorrow`方法，在对象被成功借出后更新一些统计项，例如返回对象池的对象个数等

```
//....
private final LinkedBlockingDeque<PooledObject<T>> idleObjects;
//....
public T borrowObject(final long borrowMaxWaitMillis) throws Exception {
        assertOpen();
        final AbandonedConfig ac = this.abandonedConfig;
        if (ac != null && ac.getRemoveAbandonedOnBorrow() &&
                (getNumIdle() < 2) &&
                (getNumActive() > getMaxTotal() - 3) ) {
            removeAbandoned(ac);
        }
        PooledObject<T> p = null;
        // Get local copy of current config so it is consistent for entire
        // method execution
        final boolean blockWhenExhausted = getBlockWhenExhausted();
        boolean create;
        final long waitTime = System.currentTimeMillis();
        while (p == null) {
            create = false;
            p = idleObjects.pollFirst();
            if (p == null) {
                p = create();
                if (p != null) {
                    create = true;
                }
            }
            if (blockWhenExhausted) {
                if (p == null) {
                    if (borrowMaxWaitMillis < 0) {
                        p = idleObjects.takeFirst();
                    } else {
                        p = idleObjects.pollFirst(borrowMaxWaitMillis,
                                TimeUnit.MILLISECONDS);
                    }
                }
                if (p == null) {
                    throw new NoSuchElementException(
                            "Timeout waiting for idle object");
                }
            } else {
                if (p == null) {
                    throw new NoSuchElementException("Pool exhausted");
                }
            }
            if (!p.allocate()) {
                p = null;
            }
            if (p != null) {
                try {
                    factory.activateObject(p);
                } catch (final Exception e) {
                    try {
                        destroy(p, DestroyMode.NORMAL);
                    } catch (final Exception e1) {
                        // Ignore - activation failure is more important
                    }
                    p = null;
                    if (create) {
                        final NoSuchElementException nsee = new NoSuchElementException(
                                "Unable to activate object");
                        nsee.initCause(e);
                        throw nsee;
                    }
                }
                if (p != null && getTestOnBorrow()) {
                    boolean validate = false;
                    Throwable validationThrowable = null;
                    try {
                        validate = factory.validateObject(p);
                    } catch (final Throwable t) {
                        PoolUtils.checkRethrow(t);
                        validationThrowable = t;
                    }
                    if (!validate) {
                        try {
                            destroy(p, DestroyMode.NORMAL);
                            destroyedByBorrowValidationCount.incrementAndGet();
                        } catch (final Exception e) {
                            // Ignore - validation failure is more important
                        }
                        p = null;
                        if (create) {
                            final NoSuchElementException nsee = new NoSuchElementException(
                                    "Unable to validate object");
                            nsee.initCause(validationThrowable);
                            throw nsee;
                        }
                    }
                }
            }
        }
        updateStatsBorrow(p, System.currentTimeMillis() - waitTime);
        return p.getObject();
    }
```

3.工厂的`passivateObject(PooledObject<T> p)`和`passivateObject(PooledObject<T> p)`即对象的激活和钝化方法有什么用？

如以下源码所示，在对象使用完被返回对象池时，如果校验失败直接销毁，如果校验通过需要先钝化对象再存入空闲队列。至于激活对象的方法在上述取用对象时也会先激活再被取出。

因此我们可以发现处于空闲和使用中的对象他们除了状态不一致，我们也可以通过激活和钝化的方式在他们之间增加新的差异，例如我们要做一个Elasticsearch连接池，每个对象就是一个带有ip和端口的连接实例，很显然访问es集群是多个不同的ip，所以每次访问的ip不一定相同，我们则可以在激活操作为对象赋值ip和端口，钝化操作中将ip和端口归为默认值或者空，这样流程更为标准。

```
 public void returnObject(final T obj) {
        final PooledObject<T> p = allObjects.get(new IdentityWrapper<>(obj));
      //....
        //校验失败直接销毁 return
    //...
        try {
            factory.passivateObject(p);
        } catch (final Exception e1) {
            swallowException(e1);
            try {
                destroy(p, DestroyMode.NORMAL);
            } catch (final Exception e) {
                swallowException(e);
            }
            try {
                ensureIdle(1, false);
            } catch (final Exception e) {
                swallowException(e);
            }
            updateStatsReturn(activeTime);
            return;
        }
      //......
      //返回空闲队列
    }
```

## 对象池相关配置项

对象池提供了许多配置项，在我们使用的`GenericObjectPool`默认基础对象池中可以通过构造方法传参传入`GenericObjectPoolConfig`，当然我们也可以看`GenericObjectPoolConfig`底层实现的基础类`BaseObjectPoolConfig`，具体包含如下配置:

- maxTotal：对象池中最大使用数量，默认为8
- maxIdle：对象中空闲对象最大数量，默认为8
- minIdle：对象池中空闲对象最小数量，默认为8
- lifo：当去获取对象池中的空闲实例时，是否需要遵循后进先出的原则，默认为`true`
- blockWhenExhausted：当对象池处于`exhausted`状态，即可用实例为空时，是否阻塞来获取实例的线程，默认 `true`
- fairness：当对象池处于`exhausted`状态，即可用实例为空时，大量线程在同时阻塞等待获取可用的实例，`fairness`配置来控制是否启用公平锁算法，即先到先得，默认为`false`。这一项的前提是`blockWhenExhausted`配置为`true`
- maxWaitMillis：最大阻塞时间，当对象池处于`exhausted`状态，即可用实例为空时，大量线程在同时阻塞等待获取可用的实例，如果阻塞时间超过了`maxWaitMillis`将会抛出异常。当此值为负数时，代表无限期阻塞直到可用。默认为`-1`
- testOnCreate：创建对象前是否校验（即调用工厂的`validateObject()`方法），如果检验失败，那么`borrowObject()`返回将失败，默认为`false`
- testOnBorrow：取用对象前是否检验，默认为`false`
- testOnReturn：返回对象池前是否检验，即调用工厂的`returnObject()`，若检验失败会销毁对象而不是返回池中，默认为`false`
- `timeBetweenEvictionRunsMillis`：驱逐周期，默认为`-1`代表不进行驱逐测试
- `testWhileIdle`：处于idle队列中即闲置的对象是否被驱逐器进行驱逐验证，当该对象上次运行时间距当前超过了`setTimeBetweenEvictionRunsMillis(long))`设置的值，将会被驱逐验证，调用`validateObject()`方法，若验证成功，对象将会销毁。默认为`false`

## 使用步骤

1. 创建工厂类：通过继承`BaseGenericObjectPool`或者实现基础接口`PooledObjectFactory`,并按照业务需求重写对象的创建、销毁、校验、激活、钝化方法，其中销毁多为连接的关闭、置空等。
2. 创建池：通过继承`GenericObjectPool`或者实现基础接口`ObjectPool`，建议使用前者，它为我们提供了空闲对象驱逐检测机制（即将空闲队列中长时间未使用的对象销毁，降低内存占用），以及提供了很多对象的基本信息，例如对象最后被使用的时间、使用对象前是否检验等。
3. 创建池相关配置（可选）：通过继承`GenericObjectPoolConfig`或者继承`BaseObjectPoolConfig`，来增加对线程池的配置控制，建议使用前者，它为我们实现了基本方法，只需要自己添加需要的属性即可。
4. 创建包装类（可选）：即要存在于对象池中的对象，在实际对象之外添加许多基础属性，便于了解对象池中对象的实时状态。

## 注意事项

我们虽然使用了默认实现，但是也应该结合实际生产情况进行优化，不能使用了线程池而性能却更低了。在使用中我们应注意以下事项：

- 要为对象池设置空闲队列最大最小值，默认最大最小值，默认最大为8往往不能满足需要

```
private volatile int maxIdle = GenericObjectPoolConfig.DEFAULT_MAX_IDLE;
private volatile int minIdle = GenericObjectPoolConfig.DEFAULT_MIN_IDLE;
public static final int DEFAULT_MAX_IDLE = 8;
public static final int DEFAULT_MIN_IDLE = 0;
```

- 对象池设置`maxWaitMillis`属性，即取用对象最大等待时间
- 使用完对象及时释放对象，将对象返回池中，特别是发生了异常也要通过`try..chatch..finally`的方式确保释放，避免占用资源

我们展开讲讲注意事项，首先为什么要设置`maxWaitMillis`，我们取用对象使用的如下方法

```
public T borrowObject() throws Exception {
    return borrowObject(getMaxWaitMillis());
}
```

可以看到默认的最大等待时间为`-1L`

```
private volatile long maxWaitMillis =
        BaseObjectPoolConfig.DEFAULT_MAX_WAIT_MILLIS;
 //....
public static final long DEFAULT_MAX_WAIT_MILLIS = -1L;
```

我们再来查看取用对象逻辑,`blockWhenExhausted`默认为true，意思是当池中不存在空闲对象时，又来取用对象，线程将会被阻塞直到有新的可用对象。从上我们得知`-1L`将会执行`idleObjects.takeFirst()`

```
public T borrowObject(final long borrowMaxWaitMillis) throws Exception {
        //.......
        final boolean blockWhenExhausted = getBlockWhenExhausted();
        boolean create;
        final long waitTime = System.currentTimeMillis();
        while (p == null) {
          //.......
            if (blockWhenExhausted) {
                if (p == null) {
                    if (borrowMaxWaitMillis < 0) {
                        p = idleObjects.takeFirst();
                    } else {
                        p = idleObjects.pollFirst(borrowMaxWaitMillis,
                                TimeUnit.MILLISECONDS);
                    }
                }
            }
        }
}
```

如下，阻塞队列将会一直阻塞，直到有了空闲对象才停止阻塞，这样的设定将会在吞吐提高时造成大面积阻塞影响

```
public E takeFirst() throws InterruptedException {
     lock.lock();
     try {
         E x;
         while ( (x = unlinkFirst()) == null) {
             notEmpty.await();
         }
         return x;
     } finally {
         lock.unlock();
     }
 }
```

还有一个注意事项就是要记得回收资源，即调用`public void returnObject(final T obj)`方法，原因显而易见，对象池对我们是否使用完了对象是无感知的，需要我们调用该方法回收对象，特别是发生异常也要保证回收，因此最佳实践如下：

```
 try{
   item = pool.borrowObject();
 } catch(Exception e) {
   log.error("....");
 } finally {
   pool.returnObject(item);
 }
```

## 实例使用

### 实例1：实现一个简单的字符串池

创建字符串工厂

```
package com.anqi.demo.demopool2.pool.fac;

import org.apache.commons.pool2.BasePooledObjectFactory;
import org.apache.commons.pool2.PooledObject;
import org.apache.commons.pool2.impl.DefaultPooledObject;

/**
 * 字符串池工厂
 */
public class StringPoolFac extends BasePooledObjectFactory<String> {
    public StringPoolFac() {
        super();
    }

    @Override
    public String create() throws Exception {
        return "str-val-";
    }

    @Override
    public PooledObject<String> wrap(String s) {
        return new DefaultPooledObject<>(s);
    }

    @Override
    public void destroyObject(PooledObject<String> p) throws Exception {
    }

    @Override
    public boolean validateObject(PooledObject<String> p) {
        return super.validateObject(p);
    }

    @Override
    public void activateObject(PooledObject<String> p) throws Exception {
        super.activateObject(p);
    }

    @Override
    public void passivateObject(PooledObject<String> p) throws Exception {
        super.passivateObject(p);
    }
}
```

创建字符串池

```
package com.anqi.demo.demopool2.pool;

import org.apache.commons.pool2.PooledObjectFactory;
import org.apache.commons.pool2.impl.GenericObjectPool;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;

/**
 * 字符串池
 */
public class StringPool extends GenericObjectPool<String> {
    public StringPool(PooledObjectFactory<String> factory) {
        super(factory);
    }

    public StringPool(PooledObjectFactory<String> factory, GenericObjectPoolConfig<String> config) {
        super(factory, config);
    }
}
```

测试主类

首先我们我们设置`setMaxTotal`为2，即最多有两个对象被取出使用，设置`setMaxWaitMillis`为3S，即最多被阻塞3S，我们循环取用3次，并不释放资源

```
import com.anqi.demo.demopool2.pool.fac.StringPoolFac;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class StringPoolTest {
    private static final Logger LOG = LoggerFactory.getLogger(StringPoolTest.class);

    public static void main(String[] args) {
        StringPoolFac fac = new StringPoolFac();
        GenericObjectPoolConfig<String> config = new GenericObjectPoolConfig<>();
        config.setMaxTotal(2);
        config.setMinIdle(1);
        config.setMaxWaitMillis(3000);
        StringPool pool = new StringPool(fac, config);
        for (int i = 0; i < 3; i++) {
            String s = "";
            try {
                s = pool.borrowObject();
                LOG.info("str:{}", s);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
//                if (!s.equals("")) {
//                    pool.returnObject(s);
//                }
            }
        }
    }
}
```

结果如下,在两次成功调用之后，阻塞3S，接着程序报错停止。这是因为可用资源最多为2，若不释放将会无资源可用，新来的调用者会被阻塞3S，之后报错取用失败。

```
16:18:42.499 [main] INFO com.anqi.demo.demopool2.pool.StringPoolTest - str:str-val-
16:18:42.505 [main] INFO com.anqi.demo.demopool2.pool.StringPoolTest - str:str-val-
java.util.NoSuchElementException: Timeout waiting for idle object
```

我们放开注释，释放资源后得到正常执行结果

```
16:20:52.384 [main] INFO com.anqi.demo.demopool2.pool.StringPoolTest - str:str-val-
16:20:52.388 [main] INFO com.anqi.demo.demopool2.pool.StringPoolTest - str:str-val-
16:20:52.388 [main] INFO com.anqi.demo.demopool2.pool.StringPoolTest - str:str-val-
```

全部代码地址

> https://github.com/Motianshi/alldemo/tree/master/demo-pool2

asdsada