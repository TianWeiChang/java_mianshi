## 看我72 变【Java并发编程】

大家好，面试连环炮系列，继续走起，今天给大家分享的Java异常面试连环跑。 

写公众号的宗旨就是希望每一篇文章都能给你带来帮助，真心的希望你有所收获，如果你觉得对公众号文章有什么好的建议或者有技术想探讨的，欢迎加我微信

今天分享的是Java并发编程必备的72连环炮，希望通过这种连环炮的方式，让大家更好吸收知识点，同时也是面试中出现频率非常高。

## 上帝视角

![1622030827395](E:\other\网络\assets\1622030827395.png)

废话不多说，直奔主题。

## 开始发炮

### 1、 什么是线程？

线程是操作系统能够进行运算调度的最小单位，它被包含在进程之中，是进程中的实际运作单位。程序员可以通过它进行多处理器编程，你可以使用多线程对运算密集型任务提速。比如，如果一个线程完成一个任务要`100毫秒`，那么用十个线程完成该任务只需`10毫秒`。

### 2、 线程和进程有什么区别？

一个进程是一个独立(`self contained`)的运行环境，它可以被看作一个程序或者一个应用。而线程是在进程中执行的一个任务。线程是进程的子集，一个进程可以有很多线程，每条线程并行执行不同的任务。不同的进程使用不同的内存空间，而所有的线程共享一片相同的内存空间。别把它和栈内存搞混，每个线程都拥有单独的栈内存用来存储本地数据。

### 3、 如何在Java中实现线程？

有两种创建线程的方法：一是实现`Runnable`接口，然后将它传递给Thread的构造函数，创建一个`Thread`对象；二是直接继承`Thread`类。

### 4、 用Runnable还是Thread？

这个问题是上题的后续，大家都知道我们可以通过继承Thread类或者调用`Runnable`接口来实现线程，问题是，那个方法更好呢？什么情况下使用它？

这个问题很容易回答，如果你知道Java不支持类的多重继承，但允许你调用多个接口。所以如果你要继承其他类，当然是调用`Runnable`接口好了。更多详细信息请点击这里。

### 5、 Thread 类中的start() 和 run() 方法有什么区别？

start()方法被用来启动新创建的线程，使该被创建的线程状态变为可运行状态。当你调用`run()`方法的时候，只会是在原来的线程中调用，没有新的线程启动，start()方法才会启动新线程。如果我们调用了`Thread`的run()方法，它的行为就会和普通的方法一样，直接运行`run()`方法。为了在新的线程中执行我们的代码，必须使用Thread.start()方法。

### 6、 Java中Runnable和Callable有什么不同？

`Runnable`和`Callable`都代表那些要在不同的线程中执行的任务。Runnable从JDK1.0开始就有了，Callable是在JDK1.5增加的。它们的主要区别是Callable的 call() 方法可以返回值和抛出异常，而Runnable的run()方法没有这些功能。Callable可以返回装载有计算结果的Future对象。

### 7、  Java中CyclicBarrier 和 CountDownLatch有什么不同？

`CyclicBarrier `和 `CountDownLatch` 都可以用来让一组线程等待其它线程。与 `CyclicBarrier `不同的是，`CountdownLatch `不能重新使用。

### 8、 Java内存模型是什么？

Java内存模型规定和指引Java程序在不同的内存架构、CPU和操作系统间有确定性地行为。它在多线程的情况下尤其重要。Java内存模型对一个线程所做的变动能被其它线程可见提供了保证，它们之间是先行发生关系。这个关系定义了一些规则让程序员在并发编程时思路更清晰。比如，先行发生关系确保了：

- 线程内的代码能够按先后顺序执行，这被称为程序次序规则。
- 对于同一个锁，一个解锁操作一定要发生在时间上后发生的另一个锁定操作之前，也叫做管程锁定规则。
- 前一个对`volatile`的写操作在后一个`volatile`的读操作之前，也叫`volatile`变量规则。
- 一个线程内的任何操作必需在这个线程的start()调用之后，也叫作线程启动规则。
- 一个线程的所有操作都会在线程终止之前，线程终止规则。
- 一个对象的终结操作必需在这个对象构造完成之后，也叫对象终结规则。
- 可传递性

强烈建议大家阅读《Java并发编程实践》第十六章来加深对Java内存模型的理解。

### 9、 Java中的volatile 变量是什么？

volatile是一个特殊的修饰符，只有成员变量才能使用它。在Java并发程序缺少同步类的情况下，多线程对成员变量的操作对其它线程是透明的。volatile变量可以保证下一个读取操作会在前一个写操作之后发生。线程都会直接从内存中读取该变量并且不缓存它。这就确保了线程读取到的变量是同内存中是一致的。

### 10、 什么是线程安全？Vector是一个线程安全类吗？

如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。一个线程安全的计数器类的同一个实例对象在被多个线程使用的情况下也不会出现计算失误。很显然你可以将集合类分成两组，线程安全和非线程安全的。Vector 是用同步方法来实现线程安全的, 而和它相似的`ArrayList`不是线程安全的。

### 11、 Java中什么是竞态条件？

在大多数实际的多线程应用中，两个或两个以上的线程需要共享对同一数据的存取。如果i线程存取相同的对象，并且每一个线程都调用了一个修改该对象状态的方法，将会发生什么呢？可以想象，线程彼此踩了对方的脚。根据线程访问数据的次序，可能会产生讹误的对象。这样的情况通常称为竞争条件。

### 12、 Java中如何停止一个线程？

`Java`提供了很丰富的API但没有为停止线程提供API。JDK 1.0本来有一些像`stop()`, `suspend() `和 `resume()`的控制方法，但是由于潜在的死锁威胁。因此在后续的JDK版本中他们被弃用了，之后`Java API`的设计者就没有提供一个兼容且线程安全的方法来停止一个线程。当`run() `或者 `call() `方法执行完的时候线程会自动结束，如果要手动结束一个线程，可以用`volatile` 布尔变量来退出`run()`方法的循环或者是取消任务来中断线程。

### 13、 一个线程运行时发生异常会怎样？

如果异常没有被捕获该线程将会停止执行。`Thread.UncaughtExceptionHandler`是用于处理未捕获异常造成线程突然中断情况的一个内嵌接口。当一个未捕获异常将造成线程中断的时候JVM会使用`Thread.getUncaughtExceptionHandler()`来查询线程的`UncaughtExceptionHandler`并将线程和异常作为参数传递给`handler`的`uncaughtException()`方法进行处理。

### 14、如何在两个线程间共享数据？

你可以通过共享对象来实现这个目的，或者是使用像阻塞队列这样并发的数据结构。这篇教程《Java线程间通信》(涉及到在两个线程间共享对象)用wait和notify方法实现了生产者消费者模型。

### 15、 Java中notify 和 notifyAll有什么区别？

这又是一个刁钻的问题，因为多线程可以等待单监控锁，`Java API` 的设计人员提供了一些方法当等待条件改变的时候通知它们，但是这些方法没有完全实现。notify()方法不能唤醒某个具体的线程，所以只有一个线程在等待的时候它才有用武之地。而`notifyAll()`唤醒所有线程并允许他们争夺锁确保了至少有一个线程能继续运行。

###  16、为什么wait, notify 和 notifyAll这些方法不在thread类里面？

一个很明显的原因是JAVA提供的锁是对象级的而不是线程级的，每个对象都有锁，通过线程获得。如果线程需要等待某些锁那么调用对象中的`wait()`方法就有意义了。如果`wait()`方法定义在`Thread`类中，线程正在等待的是哪个锁就不明显了。简单的说，由于wait，`notify`和`notifyAll`都是锁级别的操作，所以把他们定义在`Object`类中因为锁属于对象。

### 17、 什么是ThreadLocal变量？

`ThreadLocal`是Java里一种特殊的变量。每个线程都有一个`ThreadLocal`就是每个线程都拥有了自己独立的一个变量，竞争条件被彻底消除了。如果为每个线程提供一个自己独有的变量拷贝，将大大提高效率。首先，通过复用减少了代价高昂的对象的创建个数。其次，你在没有使用高代价的同步或者不变性的情况下获得了线程安全。

###  18、 什么是FutureTask？

在Java并发程序中FutureTask表示一个可以取消的异步运算。它有启动和取消运算、查询运算是否完成和取回运算结果等方法。只有当运算完成的时候结果才能取回，如果运算尚未完成get方法将会阻塞。一个FutureTask对象可以对调用了Callable和Runnable的对象进行包装，由于FutureTask也是调用了Runnable接口所以它可以提交给Executor来执行。

###  19、 Java中interrupted 和 isInterruptedd方法的区别？

`interrupted() `和` isInterrupted()`的主要区别是前者会将中断状态清除而后者不会。Java多线程的中断机制是用内部标识来实现的，调用`Thread.interrupt()`来中断一个线程就会设置中断标识为true。当中断线程调用静态方法`Thread.interrupted()`来检查中断状态时，中断状态会被清零。而非静态方法`isInterrupted()`用来查询其它线程的中断状态且不会改变中断状态标识。简单的说就是任何抛出`InterruptedException`异常的方法都会将中断状态清零。无论如何，一个线程的中断状态有有可能被其它线程调用中断来改变。

### 20、为什么wait和notify方法要在同步块中调用？

当一个线程需要调用对象的wait()方法的时候，这个线程必须拥有该对象的锁，接着它就会释放这个对象锁并进入等待状态直到其他线程调用这个对象上的notify()方法。同样的，当一个线程需要调用对象的notify()方法时，它会释放这个对象的锁，以便其他在等待的线程就可以得到这个对象锁。由于所有的这些方法都需要线程持有对象的锁，这样就只能通过同步来实现，所以他们只能在同步方法或者同步块中被调用。如果你不这么做，代码会抛出`IllegalMonitorStateException`异常。

### 21、 为什么你应该在循环中检查等待条件?

处于等待状态的线程可能会收到错误警报和伪唤醒，如果不在循环中检查等待条件，程序就会在没有满足结束条件的情况下退出。因此，当一个等待线程醒来时，不能认为它原来的等待状态仍然是有效的，在`notify()`方法调用之后和等待线程醒来之前这段时间它可能会改变。这就是在循环中使用wait()方法效果更好的原因，你可以在Eclipse中创建模板调用wait和notify试一试。如果你想了解更多关于这个问题的内容，推荐你阅读《Effective Java》这本书中的线程和同步章节。

### 22、 Java中的同步集合与并发集合有什么区别？

同步集合与并发集合都为多线程和并发提供了合适的线程安全的集合，不过并发集合的可扩展性更高。在Java1.5之前程序员们只有同步集合来用且在多线程并发的时候会导致争用，阻碍了系统的扩展性。Java5介绍了并发集合像`ConcurrentHashMap`，不仅提供线程安全还用锁分离和内部分区等现代技术提高了可扩展性。更多内容详见答案。

### 23、 Java中堆和栈有什么不同？

为什么把这个问题归类在多线程和并发面试题里？因为栈是一块和线程紧密相关的内存区域。每个线程都有自己的栈内存，用于存储本地变量，方法参数和栈调用，一个线程中存储的变量对其它线程是不可见的。而堆是所有线程共享的一片公用内存区域。对象都在堆里创建，为了提升效率线程会从堆中弄一个缓存到自己的栈，如果多个线程使用该变量就可能引发问题，这时volatile 变量就可以发挥作用了，它要求线程从主存中读取变量的值。

更多内容详见答案。

### 24、 什么是线程池？为什么要使用它？

创建线程要花费昂贵的资源和时间，如果任务来了才创建线程那么响应时间会变长，而且一个进程能创建的线程数有限。为了避免这些问题，在程序启动的时候就创建若干线程来响应处理，它们被称为线程池，里面的线程叫工作线程。从`JDK1.5`开始，`Java API`提供了`Executor`框架让你可以创建不同的线程池。比如单线程池，每次处理一个任务；数目固定的线程池或者是缓存线程池（一个适合很多生存期短的任务的程序的可扩展线程池）。

### 25、 如何写代码来解决生产者消费者问题？

在现实中你解决的许多线程问题都属于生产者消费者模型，就是一个线程生产任务供其它线程进行消费，你必须知道怎么进行线程间通信来解决这个问题。比较低级的办法是用wait和notify来解决这个问题，比较赞的办法是用Semaphore 或者 `BlockingQueue`来实现生产者消费者模型。

### 26、 如何避免死锁？

Java多线程中的死锁

死锁是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。这是一个严重的问题，因为死锁会让你的程序挂起无法完成任务，死锁的发生必须满足以下四个条件：

- 互斥条件：一个资源每次只能被一个进程使用。
- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
- 不剥夺条件：进程已获得的资源，在末使用完之前，不能强行剥夺。
- 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。

避免死锁最简单的方法就是阻止循环等待条件，将系统中所有的资源设置标志位、排序，规定所有的进程申请资源必须以一定的顺序（升序或降序）做操作来避免死锁。

### 27、 Java中活锁和死锁有什么区别？

这是上题的扩展，活锁和死锁类似，不同之处在于处于活锁的线程或进程的状态是不断改变的，活锁可以认为是一种特殊的饥饿。一个现实的活锁例子是两个人在狭小的走廊碰到，两个人都试着避让对方好让彼此通过，但是因为避让的方向都一样导致最后谁都不能通过走廊。简单的说就是，活锁和死锁的主要区别是前者进程的状态可以改变但是却不能继续执行。

### 28、 怎么检测一个线程是否拥有锁？

在`java.lang.Thread`中有一个方法叫`holdsLock()`，它返回true如果当且仅当当前线程拥有某个具体对象的锁。

### 29、 你如何在Java中获取线程堆栈？

对于不同的操作系统，有多种方法来获得Java进程的线程堆栈。当你获取线程堆栈时，JVM会把所有线程的状态存到日志文件或者输出到控制台。在Windows你可以使用`Ctrl + Break`组合键来获取线程堆栈，`Linux`下用`kill -3`命令。你也可以用`jstack`这个工具来获取，它对线程id进行操作，你可以用`jps`这个工具找到id。

### 30、 JVM中哪个参数是用来控制线程的栈堆栈小的

这个问题很简单，` -Xss`参数用来控制线程的堆栈大小。你可以查看JVM配置列表来了解这个参数的更多信息。

### 31、 Java中synchronized 和 ReentrantLock 有什么不同？

Java在过去很长一段时间只能通过`synchronized`关键字来实现互斥，它有一些缺点。比如你不能扩展锁之外的方法或者块边界，尝试获取锁时不能中途取消等。Java 5 通过Lock接口提供了更复杂的控制来解决这些问题。`ReentrantLock` 类实现了 Lock，它拥有与 synchronized 相同的并发性和内存语义且它还具有可扩展性。

### 32、 有三个线程T1，T2，T3，怎么确保它们按顺序执行（确保main()方法所在的线程是Java程序最后结束的线程）？

在多线程中有多种方法让线程按特定顺序执行，你可以用线程类的`join()`方法在一个线程中启动另一个线程，另外一个线程完成该线程继续执行。为了确保三个线程的顺序你应该先启动最后一个(T3调用T2，T2调用T1)，这样T1就会先完成而T3最后完成。

### 33、 Thread类中的yield方法有什么作用？

yield方法可以暂停当前正在执行的线程对象，让其它有相同优先级的线程执行。它是一个静态方法而且只保证当前线程放弃CPU占用而不能保证使其它线程一定能占用CPU，执行`yield()`的线程有可能在进入到暂停状态后马上又被执行。点击这里查看更多yield方法的相关内容。

### 34、 Java中ConcurrentHashMap的并发度是什么？

`ConcurrentHashMap`把实际map划分成若干部分来实现它的可扩展性和线程安全。这种划分是使用并发度获得的，它是`ConcurrentHashMap`类构造函数的一个可选参数，默认值为16，这样在多线程情况下就能避免争用。

### 35、Java中Semaphore是什么？

Java中的Semaphore是一种新的同步类，它是一个计数信号。从概念上讲，从概念上讲，信号量维护了一个许可集合。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个 release()添加一个许可，从而可能释放一个正在阻塞的获取者。但是，不使用实际的许可对象，`Semaphore`只对可用许可的号码进行计数，并采取相应的行动。信号量常常用于多线程的代码中，比如数据库连接池。更多详细信息请点击这里。

### 36、如果你提交任务时，线程池队列已满。会时发会生什么？

这个问题问得很狡猾，许多程序员会认为该任务会阻塞直到线程池队列有空位。事实上如果一个任务不能被调度执行那么`ThreadPoolExecutor’s submit()`方法将会抛出一个`RejectedExecutionException`异常。

### 37、 Java线程池中submit() 和 execute()方法有什么区别？

两个方法都可以向线程池提交任务，execute()方法的返回类型是void，它定义在`Executor`接口中, 而`submit()`方法可以返回持有计算结果的Future对象，它定义在`ExecutorService`接口中，它扩展了`Executor`接口，其它线程池类像`ThreadPoolExecutor`和`ScheduledThreadPoolExecutor`都有这些方法。更多详细信息请点击这里。

### 38、 什么是阻塞式方法？

阻塞式方法是指程序会一直等待该方法完成期间不做其他事情，`ServerSocket`的`accept()方`法就是一直等待客户端连接。这里的阻塞是指调用结果返回之前，当前线程会被挂起，直到得到结果之后才会返回。此外，还有异步和非阻塞式方法在任务完成前就返回。更多详细信息请点击这里。

### 39、 你对线程优先级的理解是什么？

每一个线程都是有优先级的，一般来说，高优先级的线程在运行时会具有优先权，但这依赖于线程调度的实现，这个实现是和操作系统相关的(OS dependent)。我们可以定义线程的优先级，但是这并不能保证高优先级的线程会在低优先级的线程前执行。线程优先级是一个int变量(从1-10)，1代表最低优先级，10代表最高优先级。

### 40、 什么是线程调度器(Thread Scheduler)和时间分片(Time Slicing)？

线程调度器是一个操作系统服务，它负责为`Runnable`状态的线程分配CPU时间。一旦我们创建一个线程并启动它，它的执行便依赖于线程调度器的实现。时间分片是指将可用的CPU时间分配给可用的`Runnable`线程的过程。分配CPU时间可以基于线程优先级或者线程等待的时间。线程调度并不受到Java虚拟机控制，所以由应用程序来控制它是更好的选择（也就是说不要让你的程序依赖于线程的`优先级`）。

### 41、 在多线程中，什么是上下文切换(context-switching)？

`上下文切换`是存储和恢复CPU状态的过程，它使得线程执行能够从中断点恢复执行。上下文切换是多任务操作系统和多线程环境的基本特征。

### 42、 如何在Java中创建Immutable对象？

Immutable对象可以在没有同步的情况下共享，降低了对该对象进行并发访问时的同步化开销。要创建不可变类，要实现下面几个步骤：通过构造方法初始化所有成员、对变量不要提供setter方法、将所有的成员声明为私有的，这样就不允许直接访问这些成员、在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝。

### 43、 Java中的ReadWriteLock是什么？

一般而言，读写锁是用来提升并发程序性能的锁分离技术的成果。Java中的`ReadWriteLock`是Java 5 中新增的一个接口，一个`ReadWriteLock`维护一对关联的锁，一个用于只读操作一个用于写。在没有写线程的情况下一个读锁可能会同时被多个读线程持有。写锁是独占的，你可以使用`JDK`中的`ReentrantReadWriteLock`来实现这个规则，它最多支持65535个写锁和65535个读锁。

### 44、 多线程中的忙循环是什么?

忙循环就是程序员用循环让一个线程等待，不像传统方法`wait()`, `sleep() `或` yield() `它们都放弃了CPU控制，而忙循环不会放弃CPU，它就是在运行一个空循环。这么做的目的是为了保留CPU缓存，在多核系统中，一个等待线程醒来的时候可能会在另一个内核运行，这样会重建缓存。为了避免重建缓存和减少等待重建的时间就可以使用它了。

### 45、volatile 变量和 atomic 变量有什么不同？

这是个有趣的问题。首先，volatile 变量和 atomic 变量看起来很像，但功能却不一样。`Volatile`变量可以确保先行关系，即写操作会发生在后续的读操作之前, 但它并不能保证原子性。例如用volatile修饰`count`变量那么 count++ 操作就不是原子性的。而`AtomicInteger`类提供的atomic方法可以让这种操作具有原子性如`getAndIncrement()`方法会原子性的进行增量操作把当前值加一，其它数据类型和引用变量也可以进行相似操作。

### 46、 如果同步块内的线程抛出异常会发生什么？

这个问题坑了很多Java程序员，若你能想到锁是否释放这条线索来回答还有点希望答对。无论你的同步块是正常还是异常退出的，里面的线程都会释放锁，所以对比锁接口我们更喜欢同步块，因为它不用花费精力去释放锁，该功能可以在finally block里释放锁实现。

### 47、 单例模式的双检锁是什么？

这个问题在Java面试中经常被问到，但是面试官对回答此问题的满意度仅为50%。一半的人写不出双检锁还有一半的人说不出它的隐患和Java1.5是如何对它修正的。它其实是一个用来创建线程安全的单例的老方法，当单例实例第一次被创建时它试图用单个锁进行性能优化，但是由于太过于复杂在JDK1.4中它是失败的。

### 48、 如何在Java中创建线程安全的Singleton？

这是上面那个问题的后续，如果你不喜欢双检锁而面试官问了创建`Singleton`类的替代方法，你可以利用`JVM`的类加载和静态变量初始化特征来创建Singleton实例，或者是利用枚举类型来创建`Singleton`。

### 49、 写出3条你遵循的多线程最佳实践

以下三条最佳实践大多数Java程序员都应该遵循：

- 给你的线程起个有意义的名字。

这样可以方便找bug或追踪。`OrderProcessor`,` QuoteProcessor or TradeProcessor` 这种名字比 `Thread-1.Thread-2 and Thread-3` 好多了，给线程起一个和它要完成的任务相关的名字，所有的主要框架甚至JDK都遵循这个最佳实践。

- 避免锁定和缩小同步的范围

锁花费的代价高昂且上下文切换更耗费时间空间，试试最低限度的使用同步和锁，缩小临界区。因此相对于同步方法我更喜欢同步块，它给我拥有对锁的绝对控制权。

- 多用同步类少用`wait `和 `notify`

首先，`CountDownLatch`, `Semaphore`,` CyclicBarrier` 和` Exchanger` 这些同步类简化了编码操作，而用wait和notify很难实现对复杂控制流的控制。其次，这些类是由最好的企业编写和维护在后续的JDK中它们还会不断优化和完善，使用这些更高等级的同步工具你的程序可以不费吹灰之力获得优化。

- 多用并发集合少用同步集合

这是另外一个容易遵循且受益巨大的最佳实践，并发集合比同步集合的可扩展性更好，所以在并发编程时使用并发集合效果更好。如果下一次你需要用到map，你应该首先想到用`ConcurrentHashMap`。

### 50、 如何强制启动一个线程？

这个问题就像是如何强制进行Java垃圾回收，目前还没有觉得方法，虽然你可以使用`System.gc()`来进行垃圾回收，但是不保证能成功。在Java里面没有办法强制启动一个线程，它是被线程调度器控制着且Java没有公布相关的API。

### 51、 Java中的fork join框架是什么？

`fork join`框架是JDK7中出现的一款高效的工具，Java开发人员可以通过它充分利用现代服务器上的多处理器。它是专门为了那些可以递归划分成许多子模块设计的，目的是将所有可用的处理能力用来提升程序的性能。f

`Fork join`框架一个巨大的优势是它使用了工作窃取算法，可以完成更多任务的工作线程可以从其它线程中窃取任务来执行。

### 52、Java多线程中调用wait() 和 sleep()方法有什么不同？

Java程序中wait 和 sleep都会造成某种形式的暂停，它们可以满足不同的需要。`wait()`方法用于线程间通信，如果等待条件为真且其它线程被唤醒时它会释放锁，而`sleep()`方法仅仅释放CPU资源或者让当前线程停止执行一段时间，但不会释放锁。需要注意的是，`sleep()`并不会让线程终止，一旦从休眠中唤醒线程，线程的状态将会被改变为Runnable，并且根据线程调度，它将得到执行。

###  53、什么是Thread Group？为什么不建议使用它？

`ThreadGroup`是一个类，它的目的是提供关于线程组的信息。

`ThreadGroup API`比较薄弱，它并没有比Thread提供了更多的功能。它有两个主要的功能：一是获取线程组中处于活跃状态线程的列表；二是设置为线程设置未捕获异常处理器(`ncaught exception handler`)。但在Java 1.5中Thread类也添加了`setUncaughtExceptionHandler(UncaughtExceptionHandler eh) `方法，所以`ThreadGroup`是已经过时的，不建议继续使用。

### 54、 什么是Java线程转储(Thread Dump)，如何得到它？

线程转储是一个`JVM`活动线程的列表，它对于分析系统瓶颈和死锁非常有用。有很多方法可以获取线程转储——使用`Profiler`，`Kill -3`命令，`jstack`工具等等。我们更喜欢`jstack`工具，因为它容易使用并且是JDK自带的。由于它是一个基于终端的工具，所以我们可以编写一些脚本去定时的产生线程转储以待分析。

### 55、 什么是Java Timer类？如何创建一个有特定时间间隔的任务？

`java.util.Timer`是一个工具类，可以用于安排一个线程在未来的某个特定时间执行。`Timer`类可以用安排一次性任务或者周期任务。

`java.util.TimerTask`是一个实现了`Runnable`接口的抽象类，我们需要去继承这个类来创建我们自己的定时任务并使用Timer去安排它的执行。

###  56、什么是原子操作？在Java Concurrency API中有哪些原子类(atomic classes)？

原子操作是指一个不受其他操作影响的操作任务单元。原子操作是在多线程环境下避免数据不一致必须的手段。

`int++`并不是一个原子操作，所以当一个线程读取它的值并加1时，另外一个线程有可能会读到之前的值，这就会引发错误。

在 `java.util.concurrent.atomic `包中添加原子变量类之后，这种情况才发生了改变。所有原子变量类都公开比较并设置原语（与比较并交换类似），这些原语都是使用平台上可用的最快本机结构（比较并交换、加载链接/条件存储，最坏的情况下是旋转锁）来实现的。`java.util.concurrent.atomic `包中提供了原子变量的 9 种风格（` AtomicInteger`；`AtomicLong`；`AtomicReference`；`AtomicBoolean`；原子整型；长型；引用；及原子标记引用和戳记引用类的数组形式，其原子地更新一对值）。

### 57、 Java Concurrency API中的Lock接口(Lock interface)是什么？对比同步它有什么优势？

Lock接口比同步方法和同步块提供了更具扩展性的锁操作。他们允许更灵活的结构，可以具有完全不同的性质，并且可以支持多个相关类的条件对象。

它的优势有：

- 可以使锁更公平
- 可以使线程在等待锁的时候响应中断
- 可以让线程尝试获取锁，并在无法获取锁的时候立即返回或者等待一段时间
- 可以在不同的范围，以不同的顺序获取和释放锁

###  58、 什么是Executor框架？

`Executor`框架同`java.util.concurrent.Executor` 接口在Java 5中被引入。`Executor`框架是一个根据一组执行策略调用，调度，执行和控制的异步任务的框架。

无限制的创建线程会引起应用程序内存溢出。所以创建一个线程池是个更好的的解决方案，因为可以限制线程的数量并且可以回收再利用这些线程。利用`Executor`框架可以非常方便的创建一个线程池。

### 59、Executors类是什么？

`Executors`为`Executor`，`ExecutorService`，`ScheduledExecutorService`，`ThreadFactory`和`Callable`类提供了一些工具方法。

Executors可以用于方便的创建线程池。

###  60、 什么是阻塞队列？如何使用阻塞队列来实现生产者-消费者模型？

`java.util.concurrent.BlockingQueue`的特性是：当队列是空的时，从队列中获取或删除元素的操作将会被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。

阻塞队列不接受空值，当你尝试向队列中添加空值的时候，它会抛出NullPointerException。

阻塞队列的实现都是线程安全的，所有的查询方法都是原子的并且使用了内部锁或者其他形式的并发控制。

BlockingQueue 接口是java collections框架的一部分，它主要用于实现生产者-消费者问题。

###  61、什么是Callable和Future?

`Java 5`在`concurrency`包中引入了`java.util.concurrent.Callable` 接口，它和`Runnable`接口很相似，但它可以返回一个对象或者抛出一个异常。

`Callable`接口使用泛型去定义它的返回类型。Executors类提供了一些有用的方法去在线程池中执行Callable内的任务。由于Callable任务是并行的，我们必须等待它返回的结果。`java.util.concurrent.Future`对象为我们解决了这个问题。在线程池提交Callable任务后返回了一个Future对象，使用它我们可以知道Callable任务的状态和得到Callable返回的执行结果。Future提供了get()方法让我们可以等待Callable结束并获取它的执行结果。

###  62什么是FutureTask?

`FutureTask`包装器是一种非常便利的机制，可将`Callable`转换成`Future`和`Runnable，`它同时实现两者的接口。

`FutureTask`类是Future 的一个实现，并实现了`Runnable`，所以可通过Excutor(线程池) 来执行。也可传递给Thread对象执行。如果在主线程中需要执行比较耗时的操作时，但又不想阻塞主线程时，可以把这些作业交给Future对象在后台完成，当主线程将来需要时，就可以通过`Future`对象获得后台作业的计算结果或者执行状态。

### 63、 什么是并发容器的实现？

Java集合类都是快速失败的，这就意味着当集合被改变且一个线程在使用迭代器遍历集合的时候，迭代器的next()方法将抛出`ConcurrentModificationException`异常。

并发容器：并发容器是针对多个线程并发访问设计的，在jdk5.0引入了concurrent包，其中提供了很多并发容器，如`ConcurrentHashMap`，`CopyOnWriteArrayList`等。

并发容器使用了与同步容器完全不同的加锁策略来提供更高的并发性和伸缩性，例如:在`ConcurrentHashMap`中采用了一种粒度更细的加锁机制，可以称为分段锁，在这种锁机制下，允许任意数量的读线程并发地访问map，并且执行读操作的线程和写操作的线程也可以并发的访问map，同时允许一定数量的写操作线程并发地修改map，所以它可以在并发环境下实现更高的吞吐量。

###  64、用户线程和守护线程有什么区别？

当我们在Java程序中创建一个线程，它就被称为用户线程。一个守护线程是在后台执行并且不会阻止`JVM`终止的线程。当没有用户线程在运行的时候，JVM关闭程序并且退出。一个守护线程创建的子线程依然是守护线程。

###  65、有哪些不同的线程生命周期？

当我们在Java程序中新建一个线程时，它的状态是New。当我们调用线程的start()方法时，状态被改变为Runnable。线程调度器会为`Runnable`线程池中的线程分配CPU时间并且讲它们的状态改变为`Running`。其他的线程状态还有`Waiting`，`Blocked `和`Dead`。

###  66、线程之间是如何通信的？

当线程间是可以共享资源时，线程间通信是协调它们的重要的手段。Object类中`wait()\notify()\notifyAll()`方法可以用于线程间通信关于资源的锁的状态。

###  67、为什么Thread类的sleep()和yield()方法是静态的？

Thread类的sleep()和yield()方法将在当前正在执行的线程上运行。所以在其他处于等待状态的线程上调用这些方法是没有意义的。这就是为什么这些方法是静态的。它们可以在当前正在执行的线程中工作，并避免程序员错误的认为可以在其他非运行线程调用这些方法。

###  68、如何确保线程安全？

在Java中可以有很多方法来保证线程安全——同步，使用原子类(`atomic concurrent classes`)，实现并发锁，使用volatile关键字，使用不变类和线程安全类。

###  69、同步方法和同步块，哪个是更好的选择？

同步块是更好的选择，因为它不会锁住整个对象（当然你也可以让它锁住整个对象）。同步方法会锁住整个对象，哪怕这个类中有多个不相关联的同步块，这通常会导致他们停止执行并需要等待获得这个对象上的锁。

###  70、如何创建守护线程？

使用Thread类的`setDaemon(true)`方法可以将线程设置为守护线程，需要注意的是，需要在调用`start()`方法前调用这个方法，否则会抛出`IllegalThreadStateException`异常。

### 71、线程调度策略？

(1) 抢占式调度策略

Java运行时系统的线程调度算法是抢占式的 (preemptive)。Java运行时系统支持一种简单的固定优先级的调度算法。如果一个优先级比其他任何处于可运行状态的线程都高的线程进入就绪状态，那么运行时系统就会选择该线程运行。新的优先级较高的线程抢占(preempt)了其他线程。但是Java运行时系统并不抢占同优先级的线程。换句话说，Java运行时系统不是分时的(time-slice)。然而，基于Java Thread类的实现系统可能是支持分时的，因此编写代码时不要依赖分时。当系统中的处于就绪状态的线程都具有相同优先级时，线程调度程序采用一种简单的、非抢占式的轮转的调度顺序。

(2) 时间片轮转调度策略

有些系统的线程调度采用时间片轮转(round-robin)调度策略。这种调度策略是从所有处于就绪状态的线程中选择优先级最高的线程分配一定的CPU时间运行。该时间过后再选择其他线程运行。只有当线程运行结束、放弃(yield)CPU或由于某种原因进入阻塞状态，低优先级的线程才有机会执行。如果有两个优先级相同的线程都在等待CPU，则调度程序以轮转的方式选择运行的线程。

### 72 在线程中你怎么处理不可捕捉异常？

`Thread.UncaughtExceptionHandler`是`java SE5`中的新接口，它允许我们在每一个`Thread`对象上添加一个异常处理器。

 

## 总结

也不知道说什么好，总之，点赞、在看、收藏，三连走起！