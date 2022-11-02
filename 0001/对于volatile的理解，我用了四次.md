## 对于volatile的理解，我用了四次



### 第一次理解

刚学java时，对于volatile的记忆就是：

- volatile保证可见性
- volatile防止指令重排序
- volatile不保证原子性

没过脑的背了一下，写代码的时候也没用到过，以为不重要，然后就不了了之。

### 第二次理解

一段代码引起好奇

```java
class Singleton{
    private volatile static Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

上图为比较经典的dcl（dubbo check lock）单例模式，双重if判断是为了防止多线程多次创建，但是instance属性为什么还要加个volatile关键字呢，有什么作用么？
其实它的作用主要体现在**禁止指令重排序**。

首先先理解下什么叫指令重排序？
指令重排序可以说是jvm对程序执行的一个优化，他可以保证普通的变量在方法执行的过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中写的顺序保持一致。如

```
x = 1;
y = 2;
```

这两条赋值语句之间没有依赖关系，所以在具体执行时可能会先赋值y在赋值x，发生了指令重排。
而上述DCL代码中虽然表面只有这instance = new Singleton();一条语句，但是这个赋值操作编译成字节码文件后是分为3个步骤来完成的：

1. 为对象开辟内存空间并赋默认值
2. 调用构造函数为对象赋初始值
3. 将instance引用指向刚开辟的内存地址

而程序在执行这三步时，会有可能先执行3再执行2，如果发生这种情况，线程一先将引用指向地址，还没来得及执行构造方法，线程二进来判断instance！=null 直接拿这半初始化的对象去使用，就出现了问题。
所以此处需要用volatile关键字来修饰变量，禁止指令重排序情况的发生。**那么volatile是如何做到禁止重排序的呢？**
《深入理解java虚拟机》中这样写道：

> 我们对volatile修饰的变量进行编译后发现，在赋值操作后多执行了一个“lock addl $0x0,(%esp)”,这个操作相当于一个内存屏障（Memory Barrier 或 Memory Fence，指重排序时不能把后面的指令重排序到内存屏障之前的位置）

也有别的博主这样写道：

> JMM为volatile加内存屏障有以下4种情况：
> 在每个volatile写操作的前面插入一个StoreStore屏障，防止写volatile与后面的写操作重排序。
> 在每个volatile写操作的后面插入一个StoreLoad屏障，防止写volatile与后面的读操作重排序。
> 在每个volatile读操作的后面插入一个LoadLoad屏障，防止读volatile与后面的读操作重排序。
> 在每个volatile读操作的后面插入一个LoadStore屏障，防止读volatile与后面的写操作重排序。

### 第三次理解

那么保证可见性又是指什么东东？
要想理解这可见性，需要先了解java内存模型（jmm）。学过计算机的同学都知道多核cpu中每个cpu都有自己的高速缓存，如L1,L2,L3，且每个cpu之间的缓存是隔离的，即数据不可见。而多个cpu又共享一个主内存，数据一般会从磁盘读取到主内存当中，当cpu需要处理数据时，需要从主内存读取数据到自己的缓存当中然后进行运算，运算结束后将最新数据同步回内存之中。当然这种模型也伴随这缓存一致性问题的出现。
其实java内存模型和cpu模型非常的类似：
每个线程拥有自己的工作内存，然后共享的变量会存放在主内存（jvm的内存）当中，线程之间工作内存互相隔离。如图：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJsvLs23KcWyVcPG8fL5vcOKiaAARgs4jeqM4BCJ1dyGqOBKgarFo59hg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> 上图来源于《深入理解java虚拟机363页》

我们再来看个容易理解的图：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJuobEJ34ITNoiaaZ3VoQLpszHbrhnB6qFEYj0jS8fEtibSicHHueIkKKiaw/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

再回到我们的保证可见性的探讨：
如上图所示，若线程A和B都操作主内存的共享变量时，AB会将共享变量先拷贝会自己的工作内存，在A率先完成修改完之后再同步刷回到主内存当中，此时线程B本地内存的数据还是最先拷贝的旧数据，没有及时的获取到已修改的最新数据，最后会造成数据不一致问题。
**而volatile修饰变量时，它会保证修改的值会立即被更新到主存，并通知其他线程当前缓存的变量已失效，需要重新到主内存中读取。**
底层也是通过**内存屏障**来保证的。
针对这个特性，常见的使用的场景为状态标记量

```java
public class VolatileTest1 {
    volatile static boolean flag = false;

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println("t1 start");
            while (!flag){
                System.out.println("doing something");
            }
        },"t1").start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        flag = true;
    }
}
```

和我们期望的一样，1秒后，程序正常停止。
但是好奇的我开始思考，那是不是只要不加volatile，程序就不会停止？立即更新的反义词是什么？正常情况下，线程会不会，什么时候会把修改的值写会主内存，别的线程又会什么时候会去重新读取？
带着好奇我把上诉代码中的volatile去掉，运行结果如图：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJaSh8hjLic1op3icFvhdZqYtSkzH6ybUFndDTe2bfCrfCvkZXQUR31LHw/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

没错 程序居然正常停掉了！
然后我又把while循环里的system输出去掉后，再次运行：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJEgcqLOwy74Zjm6kJaoP9L1HmeO4qstpsZ8oI45aDSDrvd7m53ricI1Q/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这次又没有停止！！
难道就是因为一句输出语句的问题么？我又尝试换成i++试试：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJNjuXXCGsMWoaVibVzNicgvo5aX42NUGmKqjVWAnBBNlYh2tYVy7M6eqQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这次也没有停止！！！
很神奇，搞得我也很懵逼！！！
我不知道是不是因为环境的原因，我用的jdk11和8，idea2020.1.2，
**个人初步猜测：不加volatile，即正常情况下，本地线程更新值后，会很快的写回主内存，而其他线程什么时候重新从主内存中读取是不确定的。**
上述while代码里面执行点稍微费时的操作（如输出，sleep 1s），都是可以停止的，如果循坏太快，它可能没时间去重新读取flag的值。
（希望有大佬看到小弟的这篇文章，并指点一二。）

### 第四次理解

那不保证原子性又是什么鬼？
原子性：保证指令不会受到线程上下文切换的影响，即一个操作不会被cpu切换所打断。
我们举一个最常见的案列来说明：
多个线程对同一个数字进行++操作：

```java
public class VolatileDemo {
    public volatile int inc = 0;

    public void increase() {
        inc++;
    }

    public static void main(String[] args) {
        final VolatileDemo test = new VolatileDemo();
        System.out.println("start");

        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    test.increase();
                }
            }).start();
        }

        while (Thread.activeCount() > 2) {
            Thread.yield();
        }
        System.out.println(test.inc);
    }
}
```

我们启动了20个线程对inc进行++操作，每个线程＋10000，理想结果应该为200000，但是实际运行结果却小于这个值，而且结果每次都不一样（可以多运行几次观察）：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJOTAmnbnGWqMyZQroLohFhKdxMsk9ELqDT42s1uMiaDXqjKh8vBthhCA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这是为什么呢，程序中inc已经加了volatile修饰，保证了线程的可见性，但是为什么结果还是会比预想的小呢？
这是因为++操作并不是简单的一步操作，即他不是原子性的，查看编译后的字节码文件，++的实际操作为：
（实事求是地说，使用字节码来分析并发问题仍然是不严谨的，因为即使编译出来只有一条字节码指令，也并不意味执行这条指令就是一个原子操作。一条字节码指令在解释执行时，解释器要运 行许多行代码才能实现它的语义。如果是编译执行，一条字节码指令也可能转化成若干条本地机器码 指令。）

```
 public void increase();
    Code:
       0: aload_0
       1: dup
       2: getfield      #2                  // Field inc:I
       5: iconst_1
       6: iadd
       7: putfield      #2                  // Field inc:I
      10: return
```

inc++操作分成了2.获取字段 5.准备常数1 6.进行加1操作 7.赋值 四步
不保证原子性，即无法确保这四步操作不会被cpu切换打断：

![图片](https://mmbiz.qpic.cn/mmbiz/OKUeiaP72uRzQ0Btgyd0l4mnxIOno88YJtSEtGsjvCALdic0jhIvdsdpckzmZDNI1eArIEUCcffLGyr9Vtp4ibHcg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如图cpu在线程1修改完之后还未写入内存时，切换到线程2，执行完了++操作，此时cpu切换回线程1又把inc=1 写回去，造成了inc的值被覆盖。
我们再看下普通的赋值操作的字节码文件 如：x=1

```
public void fun1(){
        inc = 1;
    } 
// 编译后
 public void fun1();
    Code:
       0: aload_0
       1: iconst_1
       2: putfield      #2                  // Field inc:I
       5: return
```

他没有`getfield`和add的操作，直接赋值，所以赋值操作算是原子性的。

**而synchronized是如何保证原子性的呢？**
通过字节码文件我们可以发现，用synchronized修饰真的代码块在前后会执行`monitorenter`和`monitorexit`指令，这`minitor`指令底层则是通过lock和unlock来满足原子性的，他只允许同时只有一个线程来操作资源。