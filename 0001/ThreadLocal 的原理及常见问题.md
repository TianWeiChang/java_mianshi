ThreadLocal 的原理及常见问题

ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其他线程来说无法获取到数据。Looper、ActivityThread 以及 AMS 中都用到了 ThreadLocal。

### 与 Synchronized 的比较

ThreadLocal 和 Synchronized 都用于解决多线程并发访问。可是 ThreadLocal 与 synchronized 有本质的差别。synchronized 是利用锁的机制，使变量或代码块 在某一时该仅仅能被一个线程訪问。而 ThreadLocal 为每个线程都提供了变量的副本，使得每个线程在某一时间访问到的并非同一个对象，这样就隔离了多个线程对数据的数据共享。

### 使用场景

1. 当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以采用 ThreadLocal。比如对于 Handler 来说，它需要获取当前线程的 Looper，很显然 Looper 的作用域就是线程并且不同线程具有不同的 Looper，这个时候通过 ThreadLocal 就可以轻松实现 Looper 在线程中的存取。
2. 复杂逻辑下的对象传递，比如监听器的传递，有些时候一个线程中的任务过于复杂，这可能表现为函数调用栈比较深以及代码入口的多样性，在这种情况下，我们又需要监听器能够贯穿整个线程的执行过程，这时可以采用 ThreadLocal，采用 ThreadLocal 可以让监听器作为线程内的全局对象而存在，在线程内部只要通过 get 方法就可以获取到监听器。

### 使用方法及原理

ThreadLocal 类接口很简单，只有 4 个方法，我们先来了解一下：

- void set(Object value)
  设置当前线程的线程局部变量的值。
- public Object get()
  该方法返回当前线程所对应的线程局部变量。
- public void remove()
  将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是 JDK 5.0 新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。
- protected Object initialValue()
  返回该线程局部变量的初始值，该方法是一个 protected 的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第 1 次调用 get() 或 set(Object)时才执行，并且仅执行 1 次。ThreadLocal 中的缺省实现直接返回一 个 null。

下面通过例子来说明，首先定义一个 ThreadLocal 对象，选择 Boolean 类型，如下所示

```
private ThreadLocal<Boolean> mThreadLocal = new ThreadLocal<>();
```

然后分别在主线程、子线程1和子线程2中设置和访问它的值

```
private void threadLocal() {
        mThreadLocal.set(true);
        Log.d(TAG, "[Thread#main]threadLocal=" + mThreadLocal.get());
        new Thread() {
            @Override
            public void run() {
                super.run();
                mThreadLocal.set(false);
                Log.d(TAG, "[Thread#1]threadLocal=" + mThreadLocal.get());
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                super.run();
                Log.d(TAG, "[Thread#2]threadLocal=" + mThreadLocal.get());
            }
        }.start();
    }
```

日志如下

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZbe84l7gWb1RXRkziaIY3gG7Tbbf2aZnR7HNSUVF405iakBxn6LrCq94g/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面的日志可以看出，虽然在不同的线程中访问的是同一个 ThreadLocal 对象，但是通过 ThreadLocal 获取到的值是不一样的。ThreadLocal 之所以有这样的效果，是因为不同线程访问同一个 ThreadLocal 的 get 方法，ThreadLocal 内部会从各自的线程中取出一个数组，然后在从数组中根据当前 ThreadLocal 的索引去查找出对应的 value 值，很显然，不同线程中的数组是不同的，这就是为什么通过 ThreadLocal 可以在不同的线程中维护一套数据的副本并且彼此互不干扰。

ThreadLocal 是一个泛型类型，它的定义为 public class ThreadLocal，只要弄清楚 ThreadLocal 的 get 和 set 方法就可以明白它的工作原理。

首先看 ThreadLocal 的 set 方法

```
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

在上面的 set 方法中，首先会获取当前线程，通过 getMap(Thread t) 来获取 ThreadLocalMap ，如果这个 map 不为空的话，就将 ThreadLocal 和 我们想存放的 value 设置进去，不然的话就创建一个 ThreadLocalMap 然后再进行设置。

```
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

ThreadLocalMap 是 ThreadLocal 里面的静态内部类，而每一个 Thread 都有一个对应的 ThreadLocalMap，所以 getMap 是直接返回 Thread 的成员，在 Thread 类中有一个成员专门用于存储线程的 ThreadLocal 的数据如下所示

```
public class Thread implements Runnable {
   /* ThreadLocal values pertaining to this thread. This map is maintained
    * by the ThreadLocal class.
    */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

在 threadLocals 内部有一个数组：private Entry[] table，ThreadLocal 的值就是存在这个 table 数组中

看下 ThreadLocal 的内部类 ThreadLocalMap 源码：

```
       static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;
            // 类似于map的key，value结构，key就是ThreadLocal，value就是需要隔离访问的变量
            Entry(ThreadLocal> k, Object v) {
                super(k);
                value = v;
            }
        }
        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        // 用数组保存了 Entry，因为可能有多个变量需要线程隔离访问
        private Entry[] table;
}
```

可以看到有个 Entry 内部静态类，它继承了 WeakReference，总之它记录了两个信息，一个是 ThreadLocal类型，一个是 Object 类型的值。getEntry 方法则是获取某个 ThreadLocal 对应的值，set 方法就是更新或赋值相应的 ThreadLocal 对应的值。

下面看 threadLocals 是如何使用 set 方法将 ThreadLocal 的值存储到 table 数组中的，如下所示

```
        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

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

上面分析了ThreadLocal的set方法，这里分析下它的get方法，如下所示：

```
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
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

回顾我们的 get 方法，其实就是拿到每个线程独有的 ThreadLocalMap 然后再用 ThreadLocal 的当前实例，拿到 Map 中的相应的 Entry，然后就可以拿到相应的值返回出去。当然，如果 Map 为空，还会先进行 map 的创建，初 始化等工作。

ThreadLocal 的 get() 方法的逻辑也比较清晰，它同样是取出当前线程的 threadLocals 对象，如果这个对象为 null，就调用 setInitialValue() 方法

```
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

在 setInitialValue() 方法中，将 initialValue() 的值赋给我们想要的值，默认情况下，initialValue() 的值为 null，当然也可以重写这个方法。

```
protected T initialValue() {
   return null;
}
```

从 ThreadLocal 的 set() 和 get() 方法可以看出，他们所操作的对象都是当前线程的 threalLocals 对象的 table 数组，因此在不同的线程中访问同一个 ThreadLocal 的 set() 和 get() 方法，他们对 ThreadLocal 所做的 读 / 写 操作权限仅限于各自线程的内部，这就是为什么可以在多个线程中互不干扰地存储和修改数据。

### 总结

ThreadLocal 是线程内部的数据存储类，每个线程中都会保存一个ThreadLocal.ThreadLocalMap threadLocals = null;，ThreadLocalMap 是 ThreadLocal 的静态内部类，里面保存了一个 private Entry[] table 数组，这个数组就是用来保存 ThreadLocal 中的值。通过这种方式，就能让我们在多个线程中互不干扰地存储和修改数据。

### ThreadLocal 引发的内存泄漏分析

**预备知识**
引用 Object o = new Object();
这个 o，我们可以称之为对象引用，而 new Object() 我们可以称之为在内存中产生了一个对象实例。

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZHwfQibicXWLMd6cibCFiaKYDVJEib21cMe5dWqvt7ZtWOpY9iaJDViao0dUSg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当写下 o=null 时，只是表示 o 不再指向堆中 object 的对象实例，不代表这个对象实例不存在了。

**强引用**就是指在程序代码之中普遍存在的，类似“Object obj=new Object()” 这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象实例。

**软引用**是用来描述一些还有用但并非必需的对象。对于软引用关联着的对象， 在系统将要发生内存溢出异常之前，将会把这些对象实例列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在 JDK 1.2 之后，提供了 SoftReference 类来实现软引用。

**弱引用**也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱 引用关联的对象实例只能生存到下一次垃圾收集发生之前。**当垃圾收集器工作时， 无论当前内存是否足够，都会回收掉只被弱引用关联的对象实例**。在 JDK 1.2 之 后，提供了 WeakReference 类来实现弱引用。

**虚引用**也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象 实例是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用 来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象 实例被收集器回收时收到一个系统通知。在 JDK 1.2 之后，提供了 PhantomReference 类来实现虚引用。

**内存泄漏的现象**
代码示例：

```
/**
 * 类说明：ThreadLocal造成的内存泄漏演示
 */
public class ThreadLocalOOM {
    private static final int TASK_LOOP_SIZE = 500;

    final static ThreadPoolExecutor poolExecutor
            = new ThreadPoolExecutor(5, 5,
            1,
            TimeUnit.MINUTES,
            new LinkedBlockingQueue<>());

    static class LocalVariable {
        private byte[] a = new byte[1024*1024*5];/*5M大小的数组*/
    }

    final static ThreadLocal localVariable
            = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        Object o = new Object();
        /*5*5=25*/
        for (int i = 0; i < TASK_LOOP_SIZE; ++i) {
            poolExecutor.execute(new Runnable() {
                public void run() {
                    //localVariable.set(new LocalVariable());
                    new LocalVariable();
                    System.out.println("use local varaible");
                    //localVariable.remove();
                }
            });

            Thread.sleep(100);
        }
        System.out.println("pool execute over");
    }
}
```

执行上面 ThreadLocalOOM，并将堆内存大小设 置为-Xmx256m，我们启用一个线程池，大小固定为 5 个线程

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZsIAtTFXUKFLdwRYhzpyRm9hScNtgktjmicvt2KWNjsibhWtELJxA820A/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

首先只简单的在每个任务中 new 出一个数组

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZ5Dug5pLmKxchX9vbZkrHl5TeeiaFrG2EQqydFfcsW9bDcAvhsT5ia2Bg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到内存的实际使用控制在 25M 左右：因为每个任务中会不断 new 出 一个 5M 的数组，5*5=25M，这是很合理的。

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZzuU9q5Zt3E5SEjQy99ZiamicHa1cFyJBnIaiasFK9AHk5IK6AHJdl093Q/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当我们启用了 ThreadLocal 以后：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZ3t4SWrbTJKNTVOP814BAZG1CfRAXSMKKaVnaLibib1xap9YibUmhG3sbA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZeicvOetxcicPmOKiclia5PkagngNpyRRC5KmDSKicYDEdEFJVLoBorNuvTw/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

内存占用最高升至 150M，一般情况下稳定在 90M 左右，那么加入一个 ThreadLocal 后，内存的占用真的会这么多？
于是，我们加入一行代码：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZYkOJXSx2y2Oh174qVA1ULP85Dthp6ZQF9WRLArKZ2u50R7AdicHwwicA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

再执行，看看内存情况:

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZ1FwHP2gcRZwvChkQ82DHF13bIicBOp93ITj3lbCicQk50ApwOJd6aICg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看见最高峰的内存占用也在 25M 左右，完全和我们不加 ThreadLocal 表现一样。

这就充分说明，确实发生了内存泄漏。

### 分析

根据我们前面对 ThreadLocal 的分析，我们可以知道每个 Thread 维护一个 ThreadLocalMap，这个映射表的 key 是 ThreadLocal 实例本身，value 是真正需 要存储的 Object，也就是说 ThreadLocal 本身并不存储值，它只是作为一个 key 来让线程从 ThreadLocalMap 获取 value。仔细观察 ThreadLocalMap，这个 map 是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。

因此使用了 ThreadLocal 后，引用链如图所示

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZbqmXRULTQMLF1g6kibJYTeGicibMLKwk5clhPpNECttKmUqJ7PkGA9oYQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图中的虚线表示弱引用。

这样，当把 ThreadLocal 变量置为 null 以后，没有任何强引用指向 ThreadLocal 实例，所以 ThreadLocal 将会被 GC 回收。这样一来，ThreadLocalMap 中就会出现 key 为 null 的 Entry，就没有办法访问这些 key 为 null 的 Entry 的 value，如果当前线程再迟迟不结束的话，这些 key 为 null 的 Entry 的 value 就会一直存在一条强 引用链：Thread Ref -> Thread -> ThreadLocalMap -> Entry -> value，而这块 value 永远不会被访问到了，所以存在着内存泄露。

只有当前 thread 结束以后，current thread 就不会存在栈中，强引用断开， Current Thread、Map value 将全部被 GC 回收。最好的做法是不在需要使用 ThreadLocal 变量后，都调用它的 remove()方法，清除数据。

其实考察 ThreadLocal 的实现，我们可以看见，无论是 get()、set()在某些时 候，调用了 `expungeStaleEntry` 方法用来清除 Entry 中 Key 为 null 的 Value，但是这是不及时的，也不是每次都会执行的，所以一些情况下还是会发生内存泄露。 只有 remove() 方法中显式调用了 expungeStaleEntry 方法。

**从表面上看内存泄漏的根源在于使用了弱引用，但是另一个问题也同样值得 思考：为什么使用弱引用而不是强引用？**

下面我们分两种情况讨论：

**key 使用强引用**：引用 ThreadLocal 的对象被回收了，但是 ThreadLocalMap 还持有 ThreadLocal 的强引用，如果没有手动删除，ThreadLocal 的对象实例不会被回收，导致 Entry 内存泄漏。

**key 使用弱引用**：引用的 ThreadLocal 的对象被回收了，由于 ThreadLocalMap 持有 ThreadLocal 的弱引用，即使没有手动删除，ThreadLocal 的对象实例也会被回收。value 在下一次 ThreadLocalMap 调用 set，get，remove 都有机会被回收。

比较两种情况，我们可以发现：**由于 ThreadLocalMap 的生命周期跟 Thread 一样长，如果都没有手动删除对应 key，都会导致内存泄漏，但是使用弱引用可 以多一层保障**。

因此，ThreadLocal 内存泄漏的根源是：**由于 ThreadLocalMap 的生命周期跟 Thread 一样长，如果没有手动删除对应 key 就会导致内存泄漏，而不是因为弱引用。**

**总结**

- JVM 利用设置 ThreadLocalMap 的 Key 为弱引用，来避免内存泄露。
- JVM 利用调用 remove、get、set 方法的时候，回收弱引用。
- 当 ThreadLocal 存储很多 Key 为 null 的 Entry 的时候，而不再去调用 remove、 get、set 方法，那么将导致内存泄漏。
- 使用**线程池+ ThreadLocal** 时要小心，因为这种情况下，线程是一直在不断的重复运行的，从而也就造成了 value 可能造成累积的情况。

### 错误使用 ThreadLocal 导致线程不安全

```
/**
 * 类说明：ThreadLocal的线程不安全演示
 */
public class ThreadLocalUnsafe implements Runnable {

    public static Number number = new Number(0);

    public void run() {
        //每个线程计数加一
        number.setNum(number.getNum() + 1);
        //将其存储到ThreadLocal中
        value.set(number);
        SleepTools.ms(2);
        //输出num值
        System.out.println(Thread.currentThread().getName() + "=" + value.get().getNum());
    }

    public static ThreadLocal value = new ThreadLocal() {
    };

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new ThreadLocalUnsafe()).start();
        }
    }

    private static class Number {
        public Number(int num) {
            this.num = num;
        }

        private int num;

        public int getNum() {
            return num;
        }

        public void setNum(int num) {
            this.num = num;
        }

        @Override
        public String toString() {
            return "Number [num=" + num + "]";
        }
    }
}
```

运行结果：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzqMp9MMpOy4z3icMYPD5cEZrE0NKXwY5ygA4bOW1CYyXQ5julcOR8gP6cmyAcSqp8VNeh4RHnaXaA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为什么每个线程都输出 5？难道他们没有独自保存自己的 Number 副本吗？ 为什么其他线程还是能够修改这个值？仔细考察 ThreadLocal 和 Thead 的代码， 我们发现 **ThreadLocalMap 中保存的其实是对象的一个引用**，这样的话，当有其 他线程对这个引用指向的对象实例做修改时，其实也同时影响了所有的线程持有 的对象引用所指向的同一个对象实例。这也就是为什么上面的程序为什么会输出一样的结果：**5 个线程中保存的是同一 Number 对象的引用，在线程睡眠的时候， 其他线程将 num 变量进行了修改，而修改的对象 Number 的实例是同一份，因此它们最终输出的结果是相同的**。

而上面的程序要正常的工作，应该的用法是让每个线程中的 ThreadLocal 都应该持有一个新的 Number 对象。去掉 public static Number number = new Number(0);中的 static 即可正常工作。