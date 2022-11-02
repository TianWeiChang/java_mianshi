美团面试：说一下 ThreadLocal 的原理

## 前言

上周我一位朋友侥幸通过美团一面，岗位是java后端开发工程师。美团面试官给他进行了二面。面试过程中他问了`ThreadLocal`原理（上次问线程池，这次问`ThreadLocal`，美团爸爸这么喜欢线程安全机制么），今天详细讲一讲`ThreadLocal`原理。

#### **ThreadLocal**

ThreadLocal是线程的内部存储类，可以在指定线程内存储数据。只有指定线程可以得到存储数据。

```
/**
* This class provides thread-local variables. These variables differ from 
* their normal counterparts in that each thread that accesses one (via its
* {@code get} or {@code set} method) has its own, independently initialized
* copy of the variable. {@code ThreadLocal} instances are typically private
* static fields in classes that wish to associate state with a thread (e.g.,
* a user ID or Transaction ID).
*/
```

每个线程都有一个ThreadLocalMap的实例对象，并且通过ThreadLocal管理。

```
/* ThreadLocal values pertaining to this thread. This map is maintained
* by the ThreadLocal class. */  
ThreadLocal.ThreadLocalMap threadLocals = null;
```

每个新线程都会实例化为一个`ThreadLocalMap`并且赋值给成员变量`ThreadLocals`，使用时若已经存在threadLocals则直接使用已经存在的对象。

## 应用场景 

当某些数据是以线程为作用域并且不同线程有不同数据副本时，考虑ThreadLocal。

无状态，副本变量独立后不影响业务逻辑的高并发场景。

如果如果业务逻辑强依赖于副本变量，则不适合用ThreadLocal解决。

#### **get()与set()**

set()是调用`ThreadLocalMap`的set()实现的

```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      map.set(this, value);
    else           
      createMap(t, value);
  }
  //getMap方法
  ThreadLocalMap getMap(Thread t) {
    //thred中维护了一个ThreadLocalMap
    return t.threadLocals;
  }
//createMapvoid
createMap(Thread t, T firstValue) {
//实例化一个新的ThreadLocalMap，并赋值给线程的成员变量threadLocals
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

**ThreadLocalMap**

`ThreadLocalMap`为每个Thread都维护了一个数组table，ThreadLocal确定了一个数组下标，而这个下标是value存储的对应位置。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/YscRq4LT5WuEOklVjf3JK2ibt1F2tzakibN7Y21icykGwS6l6aFOOLXGHWcBMJQRn7Gib1WN86nJW1BpHF7kAFKHiaA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`ThreadLocalMaps`是延迟构造的，因此只有在至少要放置一个条目时才创建。

```
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  table = new Entry[INITIAL_CAPACITY];
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  setThreshold(INITIAL_CAPACITY);
}
```

`ThreadLocalMap`初始化时创建了默认长度是16的Entry数组。通过hashCode与length位运算确定索引值i。

> 每个Thread都有一个ThreadLocalMap类型。相当于每个线程Thread都有一个Entry型的数组table。而一切读取过程都是通过操作这个数组table进行的。 

#### **set() 方法**

```
private void set(ThreadLocal<?> key, Object value) {
        // We don't use a fast path as with get() because it is at // least as common to use set() to create new entries as // it is to replace existing ones, in which case, a fast // path would fail more often than not.
          Entry[] tab = table;
          int len = tab.length;
          //通过&运算计算索引
          int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
              e != null;
              e = tab[i = nextIndex(i, len)]) {
              ThreadLocal<?> k = e.get();
            //如果存在key则覆盖
              if (k == key) {
                e.value = value;
                return;
              }
              if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
              }
            }
            //新建结点插入
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
          }
```

将threadLocalHashCode与长度进行位运算得到索引。

threadLocalHashCode的代码如下：

```
private final int threadLocalHashCode = nextHashCode();
 /**
    * The next hash code to be given out. Updated atomically. Starts at
    * zero.
   */    
    private static AtomicInteger nextHashCode = new AtomicInteger();
 /**
 * The difference between successively generated hash codes - turns
 * implicit sequential thread-local IDs into near-optimally spread
 * multiplicative hash values for power-of-two-sized tables.
 */    
     private static final int HASH_INCREMENT = 0x61c88647;
    /**
    * Returns the next hash code.
    */   
    private static int nextHashCode() {
      return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```

由于是static变量，`threadLocalHashCode`在每次加载threadLocal类时会重新初始化，同时会自增一次，增加`HASH_INCREMENT`（斐波那契散列乘数，通过该数散列出来的结果会比较均匀）。

static变量也称作静态变量，静态变量和非静态变量的区别是：静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。

而非静态变量是对象所拥有的，在创建对象的时候被初始化，存在多个副本，各个对象拥有的副本互不影响。static成员变量的初始化顺序按照定义的顺序进行初始化。

对于一个`ThreadLocal`来讲，他的索引值i是确定的。对于不同线程，同一个`threadlocal`对应的是不同table的同一下标，即是table[i]，不同线程之间的table是相互独立的。

####  

#### **get() 方法**

计算索引，直接取出

```
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

**remove() 方法**

```
/**
* Remove the entry for key.
*/   
private void remove(ThreadLocal<?> key) {
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);
  for (Entry e = tab[i];
      e != null;
      e = tab[i = nextIndex(i, len)]) {
      if (e.get() == key) {
        e.clear();
        expungeStaleEntry(i);
        return;
        }
      }
    }
```

## 线程隔离特性 

线程隔离特性，只有在线程内才能获取到对应的值，线程外不能访问。

- Synchronized是通过线程等待，牺牲时间来解决访问冲突
- `ThreadLocal`是通过每个线程单独一份存储空间，牺牲空间来解决冲突

## 内存泄露问题 

存在内存泄露问题，每次使用完`ThreadLocal`，都调用它的`remove()`方法，清除数据。

#### **Demo程序**

```
import java.util.concurrent.atomic.AtomicInteger;
/**
* <h3>Exper1</h3>
* <p>ThreadLocalId</p>
**/  
public class ThreadLocalId {
  // Atomic integer containing the next thread ID to be assigned
  private static final AtomicInteger nextId = new AtomicInteger(0);
 // Thread local variable containing each thread's ID
   private static final ThreadLocal <Integer> threadId = new ThreadLocal<Integer>()
   {
   @Override            
   protected Integer initialValue() {
   return nextId.getAndIncrement();
   }
   };
  // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
      return threadId.get();
    }
    public static void remove() {
      threadId.remove();
    }
  }
/**
* <h3>Exper1</h3>
* <p></p>
**/
public class ThreadLocalMain {
      private static void incrementSameThreadId(){
          try{
            for(int i=0;i<5;i++){
                System.out.println(Thread.currentThread()
                +"_"+i+",threadId:"+
                ThreadLocalId.get());
              }
          }finally {
            ThreadLocalId.remove();
          }
      }
      public static void main(String[] args) {
        incrementSameThreadId();
        new Thread(new Runnable() {
          @Override      
          public void run() {
            incrementSameThreadId();
          }
        }).start();
        new Thread(new Runnable() {
          @Override   
          public void run() {
            incrementSameThreadId();
          }
        }).start();
      }
  }
```

## 总结 

咱们玩归玩，闹归闹，别拿面试开玩笑。

`ThreadLocal`的原理在面试中几乎被问烂了。Thread的私有数据是存储在`ThreadLocalMap`，通过ThreadLoacl进行管理。要了解`ThreadLocal`的原理，最好多阅读几遍源码，尤其是`ThreadLocalMap`的源码部分。大家面试前要把知识点记牢。

**PS：防止找不到本篇文章，可以收藏点赞，方便翻阅查找哦。**

dada