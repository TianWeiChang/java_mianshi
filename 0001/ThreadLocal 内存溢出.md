

## ThreadLocal 内存溢出

在工作中，难免会用到`ThreadLocal `，在使用之前，老田在这里给大家提三个问题：

- `ThreadLocal`的原理是什么？
- `ThreadLocal`怎么就会导致内存泄漏？
- 弱引用怎么就成了`OOM`的背锅侠？
- `ThreadLocal`最佳使用方式是怎样的？

搞清楚上面三个问题，至少可以让我们避免一些不必要的Bug发生。比较年底了，再出现不必要的Bug很有可能会影响咱们年终奖（尽管很多人没年终奖，但还是有些人有的）。

我们先从代码案例开始，没代码都是耍流氓！

## 代码案例

1、首先看一下代码：模拟了一个线程数为`THREAD_LOOP_SIZE`的线程池，所有线程共享一个 ThreadLocal 变量，每一个线程执行的时候插入一个大的 List 集合，这里由于执行了`500` 次循环，也就是产生了500个线程，每一个线程都会依附一个 ThreadLocal 变量：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalOOMDemo {

    private static final int THREAD_LOOP_SIZE = 500;
    private static final int MOCK_BIG_DATA_LOOP_SIZE = 10000;

    private static ThreadLocal<List<User>> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {

        ExecutorService executorService = Executors.newFixedThreadPool(THREAD_LOOP_SIZE);

        for (int i = 0; i < THREAD_LOOP_SIZE; i++) {
            executorService.execute(() -> {
                threadLocal.set(new ThreadLocalOOMDemo().addBigList());
                Thread t = Thread.currentThread();
                System.out.println(Thread.currentThread().getName());
                
            });
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
    }

    private List<User> addBigList() {
        List<User> params = new ArrayList<>(MOCK_BIG_DATA_LOOP_SIZE);
        for (int i = 0; i < MOCK_BIG_DATA_LOOP_SIZE; i++) {
            params.add(new User("xuliugen", "password" + i, "男", i));
        }
        return params;
    }

    class User {
        private String userName;
        private String password;
        private String sex;
        private int age;

        public User(String userName, String password, String sex, int age) {
            this.userName = userName;
            this.password = password;
            this.sex = sex;
            this.age = age;
        }
    }
}
```

2、设置 JVM 参数设置最大内存为 256M，以便模拟出 OOM：

![enter image description here](https://images.gitbook.cn/a003a8a0-e3f0-11e7-8863-d3f6f40cb46f)

3、运行代码，输出结果：

![enter image description here](https://images.gitbook.cn/bc6c7490-e3f0-11e7-aa01-8f0875907fad)

可以看出，单线程池执行到第212的时候，就报了错误，出现 OOM 内存溢出错误。

4、在运行代码的时候，同时打开 JDK 工具 **jconsole** 监控内存变化（可以直接在 cmd 中输入 jconsole 回车调用）：

![这里写图片描述](https://img-blog.csdn.net/20171020195824605?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看出，上述内存一直递增到 JVM 设置的最大值(大致上)，然后抛出异常，程序退出!

5、这个实例可以很好的演示了：线程池中的每一个线程使用完 ThreadLocal 对象之后，再也不用，由于线程池中的线程不会退出，线程池中的线程的存在，同时 ThreadLocal 变量也会存在，占用内存！造成 OOM 溢出！

## ThreadLocal 为什么会内存泄漏

都说`ThreadLocal`使用不正当的使用` ThreadLocal` 造成` OOM` 的原因，下边详细的介绍一下：

1、首先看一下 `ThreadLocal `的原理图：

`Thread`、`ThreadLocal`、`ThreadLocalMap`、`Entry` 之间的关系：

![这里写图片描述](https://img-blog.csdn.net/20171020172529956?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图中描述了：一个 Thread 中只有一个 ThreadLocalMap，一个 ThreadLocalMap 中可以有多个 ThreadLocal 对象，其中一个 ThreadLocal 对象对应一个 ThreadLocalMap 中一个的 Entry（也就是说：一个 Thread 可以依附有多个 ThreadLocal 对象）。

在 ThreadLocal 的生命周期中，都存在这些引用。看下图：**实线代表强引用，虚线代表弱引用。**

![这里写图片描述](https://img-blog.csdn.net/20171020200142500?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2、ThreadLocal 的实现是这样的：每个 Thread 维护一个 `ThreadLocalMap` 映射表，这个映射表的 key 是 `ThreadLocal`实例本身，value 是真正需要存储的 Object。

3、也就是说 `ThreadLocal` 本身并不存储值，它只是作为一个 key 来让线程从 `ThreadLocalMap` 获取 value。值得注意的是图中的虚线，表示 `ThreadLocalMap` 是使用 `ThreadLocal` 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。

4、ThreadLocalMap 使用 ThreadLocal 的弱引用作为 key，如果一个 ThreadLocal 没有外部强引用来引用它，那么系统 GC 的时候，这个 ThreadLocal 势必会被回收，这样一来，ThreadLocalMap 中就会出现 key 为 null 的 Entry，就没有办法访问这些 key 为 null 的 Entry 的 value，如果当前线程再迟迟不结束的话，这些 key 为 null 的 Entry 的 value 就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄漏。

5、总的来说就是，ThreadLocal 里面使用了一个存在弱引用的 map，map 的类型是`ThreadLocal.ThreadLocalMap.` Map中的 key 为一个 threadlocal 实例。这个 Map 的确使用了**弱引用**，不过弱引用只是针对 key。每个 key 都弱引用指向 threadlocal。 当把 threadlocal 实例置为 null 以后，没有任何强引用指向 threadlocal 实例，所以 threadlocal 将会被 gc 回收。

但是，我们的 value 却不能回收，而这块 value 永远不会被访问到了，所以存在着内存泄露。因为存在一条从`current thread`连接过来的强引用。只有当前thread结束以后，`current thread`就不会存在栈中，强引用断开，Current Thread、Map value 将全部被 GC 回收。最好的做法是将调用 threadlocal 的 remove 方法，这也是等会后边要说的。

6、其实，ThreadLocalMap 的设计中已经考虑到这种情况，也加上了一些防护措施：**在 ThreadLocal 的get(),set(),remove()的时候都会清除线程 ThreadLocalMap 里所有 key 为 null 的 value**。这一点在上一节中也讲到过！

7、但是这些被动的预防措施并不能保证不会内存泄漏：

```
（1）使用static的ThreadLocal，延长了ThreadLocal的生命周期，可能导致内存泄漏。
（2）分配使用了ThreadLocal又不再调用get(),set(),remove()方法，那么就会导致内存泄漏，因为这块内存一直存在。
```

## OOM 是否是弱引用的锅？

关于Java中的四大引用请参考：

从表面上看内存泄漏的根源在于使用了弱引用。网上的文章大多着重分析 ThreadLocal 使用了弱引用会导致内存泄漏，但是另一个问题也同样值得思考：**为什么使用弱引用而不是强引用？**

我们先来看看官方文档的说法：

> To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys.
>
> 为了应对非常大和长时间的用途，哈希表使用弱引用的 key。

下面我们分两种情况讨论：

**（1）key 使用强引用**：引用的`ThreadLocal`的对象被回收了，但是`ThreadLocalMap`还持有`ThreadLocal`的强引用，如果没有手动删除，`ThreadLocal`不会被回收，导致 Entry 内存泄漏。

**（2）key 使用弱引用**：引用的 ThreadLocal 的对象被回收了，由于`ThreadLocalMap`持有`ThreadLocal`的弱引用，即使没有手动删除，`ThreadLocal`也会被回收。`value`在下一次`ThreadLocalMap`调用`set、get、remove`的时候会被清除。

比较两种情况，我们可以发现：由于`ThreadLocalMap`的生命周期跟 Thread 一样长，如果都没有手动删除对应 key，**都会**导致内存泄漏，但是**使用弱引用可以多一层保障**：**弱引用ThreadLocal不会内存泄漏，对应的 value 在下一次ThreadLocalMap调用set、get、remove的时候会被清除。**

因此，ThreadLocal 内存泄漏的根源是：**由于 ThreadLocalMap 的生命周期跟 Thread 一样长，如果没有手动删除对应 key 就会导致内存泄漏，而不是因为弱引用。**

## ThreadLocal 最佳实践

1、综合上面的分析，我们可以理解 ThreadLocal 内存泄漏的前因后果，那么怎么避免内存泄漏呢？

**答案就是**：每次使用完 ThreadLocal，都调用它的 remove() 方法，清除数据。

在使用线程池的情况下，没有及时清理 ThreadLocal，不仅是内存泄漏的问题，更严重的是可能导致业务逻辑出现问题。所以，使用 ThreadLocal 就跟加锁完要解锁一样，用完就清理。

**注意**：

并不是所有使用 ThreadLocal 的地方，都在最后 remove()，他们的生命周期可能是需要和项目的生存周期一样长的，所以要进行恰当的选择，以免出现业务逻辑错误！但首先应该保证的是 ThreadLocal 中保存的数据大小不是很大！

2、那么我们修改最开始的代码为：

取消注释：threadLocal.remove(); 结果不会出现 OOM，可以看出堆内存的变化呈现锯齿状，证明每一次 remove() 之后，ThreadLocal 的内存释放掉了！线程池中的线程的数量持续增加！

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalOOMDemo {

    private static final int THREAD_LOOP_SIZE = 500;
    private static final int MOCK_BIG_DATA_LOOP_SIZE = 10000;

    private static ThreadLocal<List<User>> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {

        ExecutorService executorService = Executors.newFixedThreadPool(THREAD_LOOP_SIZE);

        for (int i = 0; i < THREAD_LOOP_SIZE; i++) {
            executorService.execute(() -> {
                threadLocal.set(new ThreadLocalOOMDemo().addBigList());
                Thread t = Thread.currentThread();
                System.out.println(Thread.currentThread().getName());
                threadLocal.remove(); 
            });
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        executorService.shutdown();
    }

    private List<User> addBigList() {
        List<User> params = new ArrayList<>(MOCK_BIG_DATA_LOOP_SIZE);
        for (int i = 0; i < MOCK_BIG_DATA_LOOP_SIZE; i++) {
            params.add(new User("xuliugen", "password" + i, "男", i));
        }
        return params;
    }

    class User {
        private String userName;
        private String password;
        private String sex;
        private int age;

        public User(String userName, String password, String sex, int age) {
            this.userName = userName;
            this.password = password;
            this.sex = sex;
            this.age = age;
        }
    }
}
```

取消注释：`threadLocal.remove();` 结果不会出现 OOM，可以看出堆内存的变化呈现锯齿状，证明每一次remove() 之后，ThreadLocal 的内存释放掉了！线程池中的线程的数量持续增加！

![这里写图片描述](https://img-blog.csdn.net/20171020203455543?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](https://img-blog.csdn.net/20171020203514504?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

------

参考文章：

<http://www.importnew.com/22039.html>