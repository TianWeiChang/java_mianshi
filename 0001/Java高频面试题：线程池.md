Java高频面试题：线程池

大家好，今天是九月最后一天，明天就是国庆节啦，提前祝大家国庆快乐！同时，也祝愿过激越来越强大，国泰民安！

下面给大家分享一波关于线程池的高频面试题，难度系数3.5星（最高5星）。

### 1.为什么要用线程池？线程池的优势是什么？ 

 线程池主要的工作是控制运行的线程数量，处理过程中将任务放进队列里，然后在线程创建后启动这些任务，如果线程数量超过了最大数量的线程排队等候，等其他线程执行完毕，再从队列里取出任务来执行。 

 主要特点：线程复用、控制最大并发数、管理线程 

 （1）降低资源消耗，通过重复利用已创建的线程降低线程创建和销毁造成的损耗； 

 （2）提高响应速度，当任务到达时，任务可以不需要等到线程创建就可以执行； 

 （3）提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配、调优和监控。 

 线程池是一个管理线程的池子。由于创建和关闭线程需要花费时间，如果为每一个任务都创建一个线程，非常消耗资源。使用线程池可以避免增加创建和销毁线程的资源消耗，提高响应速度，且能重复利用线程。在使用线程池后，创建线程就变成了从线程池中获取空闲线程，关闭线程变成了向线程池归还线程。 

###  2.线程池的几个重要参数介绍？  

 ```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) 
 ```

- `corePoolSize`: 线程池中核心线程的数量  
- `maximumPoolSize `：线程池中最大线程数量 
- `keepAliveTime`：非核心线程的存活时间 
- `TimeUnit unit`：存活时间单位 
- `workQueue`：任务队列 
- `threadFactory`：线程工厂，用于创建线程，一般用默认的即可 
- `handler`：拒绝策略，当队列满了并且工作线程大于等于线程池的最大线程数 

### 3.线程池的执行流程？ 

（1）在创建线程池后，等待提交过来的任务请求； 

 （2）当调用execute()方法添加一个请求任务时，线程池会做如下判断： 

 如果正在运行的线程数量小于`corePoolSize`，那么马上创建核心线程运行这个任务； 

 如果正在运行的线程数量大于或者等于`corePoolSize`，那么将这个任务放入任务队列中； 

 如果任务队列满了且正在运行的线程数量小于`maximumPoolSize`(最大线程数)，那么创建一个非核心线程立刻运行这个任务； 

 如果任务队列满了且正在运行的线程数量大于或等于`maximumPoolSize`，线程池会执行拒绝策略； 

 （3）当一个线程完成任务时，会在队列中取下一个任务来执行； 

 （4）当一个线程无事可做超过一定时间时，线程池会停掉。 

### 4.拒绝策略？ 

 等待的任务队列满了，容纳不下新任务，同时线程池中的最大线程数也达到了，无法创建新的非核心线程去处理任务，此时需要拒绝策略。 

- `AbortPolicy`：抛出 `RejectedExecutionException`异常阻止系统正常进行； 
-  `CallerRunsPolicy`：将任务回退到调用者，由调用线程处理该任务。 
- `DiscardOldestPolicy`：丢弃任务队列中等待最久的任务，将当前任务放入任务队列中； 
- `DiscardPolicy`：直接丢弃任务，不处理也不抛出异常； 

### 5.单一的、固定数、可变的三中创建线程的方法，你在工作中用到过哪个？  

```java
//ExecutorService threadPool = Executors.newFixedThreadPool(5);//一池5个处理线程
//ExecutorService threadPool = Executors.newFixedThreadPool(1);//一池1个线程
ExecutorService threadPool = Executors.newCachedThreadPool();//一池N个线程
```

一般不适用这三种方法，[阿里巴巴]()Java开发手册中说过，线程池不允许使用Executors去创建，而是通过`ThreadPoolExecutor `的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。  

 说明：Executors返回的线程池对象的弊端如下： 

 （1）`FixedThreadPool`和`SingleThreadPool`

 允许请求的队列长度为`Integer.MAX_VALUE`，可能会堆积大量的请求，从而导致`OOM`； 

 （2）`CacheThreadPool和ScheduledThreadPool`: 

 允许的创建线程数量为`Integer.MAX_VALUE`，可能会创建大量的线程，从而导致`OOM`； 

### 6.如何合理配置线程池？ 

 分两种，CPU密集和IO密集 

 线程池究竟设置多大要看你的线程池执行的什么任务了，有CPU密集型和IO密集型，任务类型不同，分配的线程池大小不同。 

##### CPU密集 

 CPU密集的意思是该任务需要大量的运算，而没有阻塞，CPU一直全速运行。 

 CPU密集任务只有在真正的多核CPU上才可能得到加速(通过多线程)，而在单核CPU上，无论你开几个模拟的多线程，该任务都不可能得到加速，因为CPU总的运算能力就那些。 

 CPU密集型任务应配置尽可能小的线程，一般公式是：配置CPU核数+1个线程的线程池， 

#####  IO密集 

 IO密集型，即该任务需要大量的IO，即大量的阻塞。在单线程上运行IO密集型的任务会导致浪费大量的CPU运算能力浪费在等待。所以在IO密集型任务中使用多线程可以大大的加速程序运行，即时在单核CPU上，这种加速主要就是利用了被浪费掉的阻塞时间。 

 方法一：可以使用较大的线程池，一般CPU核心数 * 2 

 IO密集型CPU使用率不高，可以让CPU等待IO的时候处理别的任务，充分利用cpu时间 

 方法二：线程等待时间所占比例越高，需要越多线程。线程CPU时间所占比例越高，需要越少线程。 

 下面举个例子： 

 比如平均每个线程CPU运行时间为0.5s，而线程等待时间（非CPU运行时间，比如IO）为1.5s，CPU核心数为8，那么根据上面这个公式估算得到：((0.5+1.5)/0.5)8=32。这个公式进一步转化为： 

>  最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1） CPU数目。

实际项目中，这个计算方式只是一个理论值而已，具体还得多次压测，最终得出相对最优质的。

daa