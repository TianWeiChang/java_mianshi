阿里二面：main 方法可以继承吗？

昨天，微信群里一位网友，在群里发了自己面试阿里的过程。其中一个面试，他在群里 PUA 其他网友。这道面试题就是：`Java 中的 main 方法可以继承吗？`

我们一开始学习 Java 程序的时候，最先跑的一段代码肯定是 main 方法，main 方法的格式如下：

```
public static void main(String[] args) {

}
```

那么 main 方法有什么特殊的地方呢？今天我们来简单看一下。

首先针对 main 方法的格式定义：

**「public」** ：main 方法是启动的时候由 JVM 进行加载的，public 的可访问权限是最高的，所以需要声明为 public；

**「static」** ：方法的调用要么是通过对象，要么是通过类，而 main 方法的话因为是由虚拟机调用的，所以无需生成对象，那么声明为 static 即可；

**「main」** ：至于为什么方法名称叫 main，我想应该是参考的是 C 语言的方法名吧；

**「void」** ：main 方法退出时，并没有需要有相关返回值需要返回，所以是 void；

**「String[]」** ：此字符串数组用来运行时接受用户输入的参数；因为字符串在 Java 中是具有通用普遍性的，所以使用字符串是最优选择；数组的话，因为我们的参数不止一个，所以数组肯定是合适的；

不过自 JDK1.5 引入动态参数后，`String[]`数组也可以使用`String... args`来实现。

```
public static void main(String... args){

}
```

除了上面 JVM 规定的这个 main 方法比较特殊外，其他的 main 方法与普通的静态方法是没有什么不同的。

## main方法能重载么？

这个是可以的，比如说我们给它重载一个方法：

```
public class Main {
    public static void main(String args) {
        System.out.println("hello world:" + args);
    }

    public static void main(String[] args) {
        main("test");
    }
}
```

编译运行，很显然没啥问题，除了 JVM 规定的作为应用程序入口的 main 方法之外，其他的 main 方法都是比较普通的方法。

## main方法能被其他方法调用么？

```
public class Main {
    private static int times = 3;

    public static void main2(String[] args) {
        times--;
        main(args);
    }

    public static void main(String[] args) {
        System.out.println("main方法执行:" + times);
        if (times <= 0) {
            System.exit(0);
        }
        main2(args);
    }
}
```

运行一下代码，可以发现代码能正常执行：

```
main方法执行:3
main方法执行:2
main方法执行:1
main方法执行:0
```

所以说即使是作为应用程序入口的 main 方法，也是可以被其他方法调用的，但要注意程序的关闭方式，别陷入死循环了。

## main方法可以继承么？

我们以前了解过，当类继承时，子类可以继承父类的方法和变量，那么当父类定义了 main 方法，而子类没有 main 方法时，能继承父类的 main 方法，从而正常的运行程序么？

```
public class Main {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

定义子类：

```
public class Main2 extends Main {
}
```

这时候我们运行子类 Main2，可以发现，同样打印了`hello world`，这说明 main 方法也是可以继承的。那么还有一种隐藏的情况也很显然了，子类定义自己的 main 方法，隐藏掉父类中的实现，那么这也是可以的。

```
public class Main2 extends Main {
    public static void main(String [] args) {
        System.out.println("hello world Main2");
    }
}
```

这时候就会打印子类自己的内容了：`hello world Main2`。

这么来看，除了`main`方法作为应用程序的入口比较特殊外，其他情况下与正常的静态方法是没什么区别的。