## ThreadLocal 到底是什么？我们来一探究竟

### 一、前言

对一个事务的认知是一个递进的过程。在了解ThreadLocal时，需要注意以下几点：

什么是ThreadLocal？ThreadLocal出现的背景是什么？解决了什么问题？
ThreadLocal的使用方法是什么？使用的效果如何？
ThreadLocal是如何实现它的功能的，即ThreadLocal的原理是什么？

### 二、背景

在一个分布式系统中，多个线程同时访问同一类实例中的某个变量a，由于变量a是线程共享的，导致一个线程对变量a进行修改，其他线程读到的都是修改后的变量a的值（如果是存在多个线程同时写，需要加分布式锁，限制同时只能一个线程对其进行修改）。这种情况在普通的场景下是合理的，比如在电商系统中，买家点击支付订单两次（两个独立的线程），第一次生成订单，会修改幂等值（防止第二次重复下单），第二次访问的时候去判断幂等值，如果已被修改，则不会重新生成订单。所以线程间变量共享是必须的。

但存在这样一个场景：还是以电商系统为例。买家在访问订单详情页的时候，在不同的条件下会查订单（查数据库）。查库涉及io，对系统的开销和响应时间有较大的影响。由于订单详情页的渲染都是一些读操作，没有写操作，所以，需要在查数据库时做一层本地缓存。而且这个本地缓存是对线程敏感的，只在当前线程生效，别的线程无法访问这个缓存，也就是说线程间是隔离的。

上面只是以电商为例， 所以需要一种方式，能够实现变量的线程间隔离，此变量只能在当前线程生效，不同的线程变量有不同的值。基于以上诉求，java诞生了ThreadLocal，主要是为了解决内存的线程隔离。

### 三、使用方式

#### 3.1 测试代码

线程类

```java
public class NormalThread implements Runnable {

    private int shareValue = 0;
    ThreadLocal<Integer> threadLocalValue = new ThreadLocal<>();

    @Override
    public void run() {

        shareValue += 1;
        threadLocalValue.set(shareValue);
        System.out.println(Thread.currentThread().getName() + "===== shareValue:" + shareValue + "   threadLocal:" + threadLocalValue.get());
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "===== shareValue:" + shareValue + "   threadLocal:" + threadLocalValue.get());
    }
}
```

线程池

```java
public class MutiThreadUtil {

    private static int core_pool_size = 4;
    private static int max_pool_size = 10;
    //如果空闲立即退出
    private static long keep_alive_time = 0L;
    //队列的容量是0
    private static BlockingQueue queue = new SynchronousQueue();
    //队列容量为1
    private static ArrayBlockingQueue<Integer> arrayBlockingQueue = new ArrayBlockingQueue(1);

    public static ExecutorService initThreadPool() {
        ExecutorService executorService = new ThreadPoolExecutor(
            core_pool_size, max_pool_size, keep_alive_time,TimeUnit.SECONDS,queue
        );
        return executorService;
    }
}
```

主线程

```java
public class ThreadLocalTest {

    public static void main(String[] args) {

        NormalThread normalThread = new NormalThread();
        ExecutorService executorService = MutiThreadUtil.initThreadPool();
        for (int i = 0; i < 10; i ++) {
            executorService.execute(normalThread);
        }
        executorService.shutdown();
    }
}
```

#### 3.2 结果分析

```
pool-1-thread-1===== shareValue:1   threadLocal:1
pool-1-thread-2===== shareValue:2   threadLocal:2
pool-1-thread-3===== shareValue:3   threadLocal:3
pool-1-thread-8===== shareValue:4   threadLocal:4
pool-1-thread-4===== shareValue:5   threadLocal:5
pool-1-thread-5===== shareValue:6   threadLocal:6
pool-1-thread-6===== shareValue:7   threadLocal:7
pool-1-thread-7===== shareValue:8   threadLocal:8
pool-1-thread-9===== shareValue:9   threadLocal:9
pool-1-thread-10===== shareValue:10   threadLocal:10
pool-1-thread-3===== shareValue:10   threadLocal:3
pool-1-thread-2===== shareValue:10   threadLocal:2
pool-1-thread-1===== shareValue:10   threadLocal:1
pool-1-thread-8===== shareValue:10   threadLocal:4
pool-1-thread-7===== shareValue:10   threadLocal:8
pool-1-thread-4===== shareValue:10   threadLocal:5
pool-1-thread-5===== shareValue:10   threadLocal:6
pool-1-thread-9===== shareValue:10   threadLocal:9
pool-1-thread-10===== shareValue:10   threadLocal:10
pool-1-thread-6===== shareValue:10   threadLocal:7
```

以线程pool-1-thread-1线程（后面简称线程1）作为分析，刚开始 线程1的shareValue 和 threadlocal 值均为1， shareValue是共享变量，在线程1 sleep阶段，线程2-10均执行了以下代码

```java
 shareValue += 1;
 threadLocalValue.set(shareValue);
```

但从sleep后的打印结果来看，线程1只是更改了shareValue10的值，变为10， 而threadlocal的值没有变，还是1，这说明threadlocal的值是线程级的，是线程的私有空间，不会因为其他线程的改变而改变。证明了thread的线程隔离性。

### 四、ThreadLocal 原理

在不看源码之前，我们思考下如果让我们设计这样一个工具类，能够使得线程间的变量相互隔离，我们会怎样设计？
每一个线程，其执行均是依靠Thread类的实例的start方法来启动线程，然后CPU来执行线程。每一个Thread类的实例的运行即为一个线程。若要每个线程（每个Thread实例）的变量空间隔离，则需要将这个变量的定义声明在Thread这个类中。这样，每个实例都有属于自己的这个变量的空间，则实现了线程的隔离。事实上，ThreadLocal的源码也是这样实现的。

#### 4.1 实现内存线程间隔离的原理

在Thread类中声明一个公共的类变量ThreadLocalMap，用以在Thread的实例中预占空间

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

在`ThreadLocal`中创建一个内部类`ThreadLocalMap`，这个Map的key是`ThreadLocal`对象，value是set进去的`ThreadLocal`中泛型类型的值。

```java
private void set(ThreadLocal<?> key, Object value) {...}
```

在new ThreadLocal时，只是简单的创建了个ThreadLocal对象，与线程还没有任何关系，真正产生关系的是在向ThreadLocal对象中set值得时候：
1.首先从当前的线程中获取ThreadLocalMap，如果为空，则初始化当前线程的ThreadLocalMap
2.然后将值set到这个Map中去，如果不为空，则说明当前线程之前已经set过ThreadLocal对象了。
这样用一个ThreadHashMap来存储当前线程的若干个可以线程间隔离的变量，key是ThreadLocal对象，value是要存储的值（类型是ThreadLocal的泛型）

```java
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

从`ThreadLocal`中获取值 ：还是先从当前线程中获取`ThreadLocalMap`，然后使用`ThreadLocal`对象(key)去获取这个对象对应的值(value)

```java
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

到这里，如果仅仅是理解ThreadLocal是如何实现的线程级别的隔离已经完全足够了。简单的讲，就是在Thread的类中声明了ThreadLocalMap这个类，然后在使用ThreadLocal对象set值的时候将当前线程（Thread实例）进行map初始化，并将Threadlocal对应的值塞进map中，下次get的时候，也是使用这个ThreadLcoal的对象（key）去从当前线程的map中获取值（value）就可以了

#### 4.2 ThreadLocalMap的深究

从源码上看，ThreadLocalMap虽然叫做Map，但和我们常规理解的Map不太一样，因为这个类并没有实现Map这个接口，只是定义在ThreadLocal中的一个静态内部类。只是因为在存储的时候也是以key-value的形式作为方法的入参暴露出去，所以称为map。

```java
static class ThreadLocalMap {...}
//ThreadLocalMap的创建，在使用ThreadLocal对象set值的时候，会创建ThreadLocalMap的对象，可以看到，
//入参就是KV，key是ThreadLocal对象，value是一个Entry对象，存储kv（HashMap是使用Node作为KV对象存储）。
//Entry的key是ThreadLocal对象，vaule是set进去的具体值。
 void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

继续看看在创建ThreadLocalMap实例的时候做了什么？其实`ThreadLocalMap`存储是一个Entry类型的数组，key提供了hashcode用来计算存储的数组地址（散列法解决冲突），创建Entry数组（初始容量16）。
然后获取到key（ThreadLocal对象）的hashcode(是一个自增的原子int型)

```java
private final int threadLocalHashCode = nextHashCode();

private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
private static AtomicInteger nextHashCode =    new AtomicInteger();
```

使用【hashcode 模(%) 数组长度】的方式得到要将key存储到数组的哪一位。
设置数组的扩容阈值，用以后续扩容

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```

创建ThreadLcoalMap对象只有在当前线程第一次插入kv的时候发生，如果是第二次插入kv，则会进行第三步

这个set的过程其实就是根据ThreadLocal的hashcode来计算存储在Entry数组的位置
利用ThreadLocal的【hashcode 模(%) 数组长度】的方式获取存储在数组的位置
如果当前位置已存在值，则向右移一位，如果也存在值，则继续右移，直到有空位置出现为止
将当前的value存储上面两部得到的索引位置（上面这两步就是散列法的实现）
校验是否扩容，如果当前数组的中存储的值得数量大于阈值（数组长度的2/3），则扩容一倍，并将原来的数组的值重新hash至新数组中（这个过程其实就是HashMap的扩容过程）

```java
private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
}
```

#### 4.3 ThreadLocalMap和HashMap的比较

上述的整个过程其实和HashMap的实现方式很相像，相同点：
两个map都是最终用数组作为存储结构，使用key做索引，value是真正存储在数组索引上的值。
不同点：解决key冲突的方式
map解决冲突的方式不一样，HashMap采用链表法，ThreadLocalMap采用散列法（又称开放地址法）
思考：为什么不采用HashMap作为ThreadLocal的存储结构？
个人理解：

引入链表，徒增了数据结构的复杂度，并且链表的读取效率较低
更加灵活。包括方法的定义和数组的管理，更加适合当前场景
不需要HashMap的额外的很多方法和变量，需要一个更加纯粹和干净map，来存储自己需要的值，减少内存的损耗。

#### 4.4 ThreadLocal的生命周期

ThreadLocal的生命周期和当前Thread的生命周期强绑定

**正常情况**
正常情况下（当然会有非正常情况），在线程退出的时候会将threadLocals这个变量置为null，等待JVM去自动回收。
注意：Thread这个方法只是用以系统能够显示的调用退出线程，线程在结束的时候是不会调用这个方法，启动的线程是非守护线程，会在线程结束的时候由jvm自动进行空间的释放和回收。

```java
private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
}
```

**非正常情况**
由于现在多线程一般都是由线程池管理，而线程池的线程一般都是复用的，这样会导致线程一直存活，而如果使用ThreadLocal大量存储变量，会使得空间开始膨胀

**启发**
需要自己来管理ThreadLocal的生命周期，在ThreadLocal使用结束以后及时调用remove（）方法进行清理。

### 五、注意事项

注意管理ThreadLocal的声明周期，及时调用remove方法进行空间释放
注意ThreadLocal的使用方式，如果在使用中发现没有获取到预期的值，只能是自己的使用方式不对，导致获取的不是同一线程下的ThreadLocal值。