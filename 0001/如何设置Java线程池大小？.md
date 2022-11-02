如何设置Java线程池大小？

## 线程的概念

### 线程

线程是调度 CPU 资源的最小单位， 线程模型粉丝 KLT 模型与 ULT 模型，JVM 使用的是 KLT 模型， Java线程与 OS线程保持 1:1 的映射关系，也就是说有一个 java 线程也会在操作系统中有一个对应的线程， Java 线程有多种生命状态

```java
// 线程状态枚举
public enum State {
  NEW, // 创建，调用 start()

  RUNNABLE, // 就绪

  BLOCKED, // 阻塞， 竞争没有拿到锁

  WAITING, // 等待

  TIMED_WAITING, // 超时等待

  TERMINATED; // 终止
}
```

### 协程

协程，Java的设计者们也没能为开发人员在API层面提供对协程的支持。因此，如果我们想在程序中使用协程来提升特定场景下应用程序的执行性能则只能依赖于第三方开源构件（比如：quasar]、 coroutines **、** kilim等），或是采用混合编程（比如：Kotlin）的方式进行。

## Java 线程池

线程池（Thread Pool）是一种基于池化思想管理线程的工具，经常出现在多线程服务器中，如MySQL。

线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销等等，同时也降低了计算机的整体性能。线程池维护多个线程，等待监督管理者分配可并发执行的任务。这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。

### 线程池优点

合理地使用线程池能够带来3个好处：

1. 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源， 还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用 线程池，必须对其实现原理了如指掌。

### 创建参数

我们可以通过ThreadPoolExecutor来创建一个线程池。它的构造方法如下：

```
new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime,
   milliseconds,runnableTaskQueue, handler);
```

创建一个线程池时需要输入如下几个参数：

**corePoolSize(线程池的基本大小)** : 当提交一个任务到线程池时，线程池会创建一个线 程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任 务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法， 线程池会提前创建并启动所有基本线程。

**runnableTaskQueue(任务队列)** :用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列。

1. ArrayBlockingQueue: 是一个基于数组结构的有界阻塞队列，此队列按FIFO(先进先出)原 则对元素进行排序。
2. LinkedBlockingQueue: 一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通 常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列
3. SynchronousQueue: 一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用 移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工 厂方法Executors.newCachedThreadPool使用了这个队列。
4. PriorityBlockingQueue: 一个具有优先级的无限阻塞队列。

**maximumPoolSize(线程池最大数量):** 线程池允许创建的最大线程数。如果队列满了，并 且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如 果使用了无界的任务队列这个参数就没什么效果。

**ThreadFactory:** 用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设 置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线 程设置有意义的名字，代码如下。
 new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();

**RejectedExecutionHandler(饱和策略):** 当队列和线程池都满了，说明线程池处于饱和状 态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法 处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。

1. AbortPolicy: 直接抛出异常。
2. CallerRunsPolicy: 只用调用者所在线程来运行任务。
3. DiscardOldestPolicy: 丢弃队列里最近的一个任务，并执行当前任务。
4. DiscardPolicy: 不处理，丢弃掉。

当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录 日志或持久化存储不能处理的任务。

**keepAliveTime(线程活动保持时间):** 线程池的工作线程空闲后，保持存活的时间。所以， 如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。

**TimeUnit(线程活动保持时间的单位):** 可选的单位有天(DAYS)、小时(HOURS)、分钟 (MINUTES)、毫秒(MILLISECONDS)、微秒(MICROSECONDS，千分之一毫秒)和纳秒(NANOSECONDS，千分之一微秒)。

## 工作原理

通过上面几个参数的描述梳理，让我们大致了解了如何初始化一个线程池，对于线程池任务的提交和执行如下图所示：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/441316ee41084647b9b3d3ca71d7cd4d~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

## 代码示例

下面是一个创建线程池和调用的的简单例子，供大家参考：

```java
// 创建线程池
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolTest {

    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
                2, 4, 0, TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(10),
                new ThreadFactory() {
                    int i = 0;

                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r);
                        thread.setName("pool-test-" + i++);
                        return thread;
                    }
                },
                // CallerRunsPolicy 如果当前线程池没有关闭会调用当前提交任务的线程参与执行
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        for (int i = 0; i < 100; i++) {
            threadPool.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    String name = Thread.currentThread().getName();
                    if ("main".equals(name)) {
                        // CallerRunsPolicy 策略 提交任务线程参与执行
                        System.out.println("this is a idea debug line");
                    }
                    System.out.println(name + ": " + this.toString());
                }
            });
        }
        threadPool.shutdown();
        while (true) {
            if (threadPool.isTerminated()) {
                System.out.println("所有的子线程都结束了！");
                break;
            }
            Thread.sleep(100);
        }
    }
}
```

## 线程数设置参考

CPU 密集型： CPU 核数  + 1

IO 密集型： 2 倍 CPU 核数 + 1

