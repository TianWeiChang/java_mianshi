面试官：说说synchronized的底层实现原理

**synchronized 的基本使用**

synchronized 的作用主要有三个：

- **确保线程互斥的访问同步代码**
- **保证共享变量的修改能够及时可见**
- **有效解决重排序问题**

从语法上讲，synchronized 总共有三种用法：

- **修饰普通方法**
- **修饰静态方法**
- **修饰代码块**

接下来我就通过几个例子程序来说明一下这三种使用方式（为了便于比较，三段代码除了 synchronized 的使用方式不同以外，其他基本保持一致）。

## 没有同步的情况

代码段 1：

```
package com.paddx.test.concurrent;

public class SynchronizedTest {
  public void method1(){

    System.out.println("Method 1 start");
    try {
      System.out.println("Method 1 execute");
      Thread.sleep(3000);
    } catch (InterruptedException e) {
       e.printStackTrace();
    }
    System.out.println("Method 1 end");
  }

  public void method2(){
    System.out.println("Method 2 start");
    try {
      System.out.println("Method 2 execute");
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 2 end");
  }

  public static void main(String[] args) {
    final SynchronizedTest test = new SynchronizedTest();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method1();
      }
    }).start();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method2();
      }
    }).start();
  }
}
```

执行结果如下，线程 1 和线程 2 同时进入执行状态，线程 2 执行速度比线程 1 快，所以线程 2 先执行完成。 

这个过程中线程 1 和线程 2 是同时执行的：

```
Method 1 start
Method 1 execute
Method 2 start
Method 2 execute
Method 2 end
Method 1 end
```

## 对普通方法同步

代码段 2：

```
package com.paddx.test.concurrent;

public class SynchronizedTest {
  public synchronized void method1(){
    System.out.println("Method 1 start");
    try {
      System.out.println("Method 1 execute");
      Thread.sleep(3000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 1 end");
  }

  public synchronized void method2(){
    System.out.println("Method 2 start");
    try {
      System.out.println("Method 2 execute");
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 2 end");
  }

  public static void main(String[] args) {
    final SynchronizedTest test = new SynchronizedTest();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method1();
      }
    }).start();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method2();
      }
    }).start();
  }
}
```

执行结果如下，跟代码段 1 比较，可以很明显的看出，线程 2 需要等待线程 1 的 Method1 执行完成才能开始执行 Method2 方法。

```
Method 1 start
Method 1 execute
Method 1 end
Method 2 start
Method 2 execute
Method 2 end
```

## 静态方法（类）同步

代码段 3：

```
package com.paddx.test.concurrent;

public class SynchronizedTest {
  public static synchronized void method1(){
    System.out.println("Method 1 start");
    try {
      System.out.println("Method 1 execute");
      Thread.sleep(3000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 1 end");
  }

  public static synchronized void method2(){
    System.out.println("Method 2 start");
    try {
      System.out.println("Method 2 execute");
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 2 end");
  }

  public static void main(String[] args) {
    final SynchronizedTest test = new SynchronizedTest();
    final SynchronizedTest test2 = new SynchronizedTest();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method1();
      }
    }).start();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test2.method2();
      }
    }).start();
  }
 }
```

执行结果如下，对静态方法的同步本质上是对类的同步（静态方法本质上是属于类的方法，而不是对象上的方法）。

所以即使 Test 和 Test2 属于不同的对象，但是它们都属于 SynchronizedTest 类的实例。

所以也只能顺序的执行 Method1 和 Method2，不能并发执行：

```
Method 1 start
Method 1 execute
Method 1 end
Method 2 start
Method 2 execute
Method 2 end
```

## 代码块同步

代码段 4：

```
package com.paddx.test.concurrent;

public class SynchronizedTest {
  public void method1(){
    System.out.println("Method 1 start");
    try {
      synchronized (this) {
        System.out.println("Method 1 execute");
        Thread.sleep(3000);
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 1 end");
  }

  public void method2(){
    System.out.println("Method 2 start");
    try {
      synchronized (this) {
        System.out.println("Method 2 execute");
        Thread.sleep(1000);
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println("Method 2 end");
  }

  public static void main(String[] args) {
    final SynchronizedTest test = new SynchronizedTest();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method1();
      }
    }).start();

    new Thread(new Runnable() {
      @Override
      public void run() {
        test.method2();
      }
    }).start();
  }
}
```

执行结果如下，虽然线程 1 和线程 2 都进入了对应的方法开始执行，但是线程 2 在进入同步块之前，需要等待线程 1 中同步块执行完成。

```
Method 1 start
Method 1 execute
Method 2 start
Method 1 end
Method 2 execute
Method 2 end
```

## Synchronized 原理

如果对上面的执行结果还有疑问，也先不用急，我们先来了解 synchronized 的原理。 

再回头上面的问题就一目了然了。我们先通过反编译下面的代码来看看 synchronized 是如何实现对代码块进行同步的：

```
package com.paddx.test.concurrent;
public class SynchronizedMethod {
  public synchronized void method() {
    System.out.println("Hello World!");
  }
}
```

反编译结果：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/vnOqylzBGCSGYBrVlwtwbdRalrYnApffTlmg1U6HlgqV97kPM3uNo9OBFyaQ3OY8rHvT83PS9viaM6AY6y1njZA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

关于这两条指令的作用，我们直接参考 JVM 规范中描述：

**monitorenter ：**Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:

• If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.

• If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.

• If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

这段话的大概意思为：每个对象有一个监视器锁（Monitor），当 Monitor 被占用时就会处于锁定状态。

线程执行 Monitorenter 指令时尝试获取 Monitor 的所有权，过程如下：

- 如果 Monitor 的进入数为 0，则该线程进入 Monitor，然后将进入数设置为 1，该线程即为 Monitor 的所有者。
- 如果线程已经占有该 Monitor，只是重新进入，则进入 Monitor 的进入数加 1。
- 如果其他线程已经占用了 Monitor，则该线程进入阻塞状态，直到 Monitor 的进入数为 0，再重新尝试获取 Monitor 的所有权。

**monitorexit：**The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.

The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. 

Other threads that are blocking to enter the monitor are allowed to attempt to do so.

**这段话的大概意思为：**执行 Monitorexit 的线程必须是 Objectref 所对应的 Monitor 的所有者。

指令执行时，Monitor 的进入数减 1，如果减 1 后进入数为 0，那线程退出 Monitor，不再是这个 Monitor 的所有者。

其他被这个 Monitor 阻塞的线程可以尝试去获取这个 Monitor 的所有权。

通过这两段描述，我们应该能很清楚的看出 Synchronized 的实现原理。

synchronized 的语义底层是通过一个 Monitor 的对象来完成，其实 Wait/Notify 等方法也依赖于 Monitor 对象。

这就是为什么只有在同步的块或者方法中才能调用 Wait/Notify 等方法，否则会抛出 `java.lang.IllegalMonitorStateException `的异常。

我们再来看一下同步方法的反编译结果，源代码如下：

```
package com.paddx.test.concurrent;

public class SynchronizedMethod {
  public synchronized void method() {
    System.out.println("Hello World!");
  }
}
```

反编译结果：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/vnOqylzBGCSGYBrVlwtwbdRalrYnApffUV9veibIURcCjMH2dEpI2gia47nddV5qGSuJFMkpVu5EhFicfIf8bc7zA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

从反编译的结果来看，方法的同步并没有通过指令 Monitorenter 和 Monitorexit 来完成（理论上其实也可以通过这两条指令来实现）。不过相对于普通方法，其常量池中多了 ACC_SYNCHRONIZED 标示符。

JVM 就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置。

如果设置了，执行线程将先获取 Monitor，获取成功之后才能执行方法体，方法执行完后再释放 Monitor。在方法执行期间，其他任何线程都无法再获得同一个 Monitor 对象。

其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

**运行结果解释**

有了对 synchronized 原理的认识，再来看上面的程序就可以迎刃而解了。

**①代码段 2 结果**

虽然 Method1 和 Method2 是不同的方法，但是这两个方法都进行了同步，并且是通过同一个对象去调用的。

所以调用之前都需要先去竞争同一个对象上的锁（Monitor），也就只能互斥的获取到锁，因此，Method1 和 Method2 只能顺序的执行。

**②代码段 3 结果**

虽然 Test 和 Test2 属于不同对象，但是 Test 和 Test2 属于同一个类的不同实例。

由于 Method1 和 Method2 都属于静态同步方法，所以调用的时候需要获取同一个类上 Monitor（每个类只对应一个 Class 对象），所以也只能顺序的执行。

**③代码段 4 结果**

对于代码块的同步，实质上需要获取 synchronized 关键字后面括号中对象的 Monitor。

由于这段代码中括号的内容都是 This，而 Method1 和 Method2 又是通过同一的对象去调用的，所以进入同步块之前需要去竞争同一个对象上的锁，因此只能顺序执行同步块。

## 总结

synchronized 是 Java 并发编程中最常用的用于保证线程安全的方式，其使用相对也比较简单。

但是如果能够深入了解其原理，对监视器锁等底层知识有所了解，一方面可以帮助我们正确的使用 synchronized 关键字。

另一方面也能够帮助我们更好的理解并发编程机制，有助于我们在不同的情况下选择更优的并发策略来完成任务。对平时遇到的各种并发问题，也能够从容的应对。