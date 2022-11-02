面试官问我 StringBuilder 线程不安全的点在哪儿

## 引言

**面试官：** **`StringBuilder`和`StringBuffer`的区别在哪？**

我：`StringBuilder`不是线程安全的，`StringBuffer`是线程安全的

**面试官：** **那`StringBuilder`不安全的点在哪儿？**

我：。。。（哑巴了）

在这之前我只记住了`StringBuilder`不是线程安全的，`StringBuffer`是线程安全的这个结论，至于`StringBuilder`为什么不安全从来没有去想过。

## 分析

在分析这个问题之前我们要知道`StringBuilder`和`StringBuffer`的内部实现跟String类一样，都是通过一个char数组存储字符串的，不同的是String类里面的char数组是final修饰的，是不可变的，而`StringBuilder`和`StringBuffer`的`char数组`是可变的。

首先通过一段代码去看一下多线程操作`StringBuilder`对象会出现什么问题

```java
public class StringBuilderDemo {  
  
    public static void main(String[] args) throws InterruptedException {  
        StringBuilder stringBuilder = new StringBuilder();  
        for (int i = 0; i < 10; i++){  
            new Thread(new Runnable() {  
                @Override  
                public void run() {  
                    for (int j = 0; j < 1000; j++){  
                        stringBuilder.append("a");  
                    }  
                }  
            }).start();  
        }  
  
        Thread.sleep(100);  
        System.out.println(stringBuilder.length());  
    }  
  
}  
```

我们能看到这段代码创建了10个线程，每个线程循环1000次往StringBuilder对象里面append字符。正常情况下代码应该输出10000，但是实际运行会输出什么呢？

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueq8rlWFnejuWibbkDsLW8SfuZtk1Rqmrob3xwxherGBMIg58MoBGLcXaqTxOu6eoVrFH9M5NnQFmw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们看到输出了“9326”，小于预期的10000，并且还抛出了一个`ArrayIndexOutOfBoundsException`异常（异常不是必现）。

### 1、为什么输出值跟预期值不一样

我们先看一下`StringBuilder`的两个成员变量（这两个成员变量实际上是定义在`AbstractStringBuilder`里面的，`StringBuilder`和`StringBuffer`都继承了`AbstractStringBuilder`）

```java
//存储字符串的具体内容  
char[] value;  
//已经使用的字符数组的数量  
int count;  
```

再看`StringBuilder`的`append()`方法：

```java
@Override  
public StringBuilder append(String str) {  
    super.append(str);  
    return this;  
}  
```

StringBuilder的append()方法调用的父类AbstractStringBuilder的append()方法

```java
public AbstractStringBuilder append(String str) {  
    if (str == null)  
        return appendNull();  
    int len = str.length();  
    ensureCapacityInternal(count + len);  
    str.getChars(0, len, value, count);  
    count += len;  
    return this;  
}  
```

我们先不管代码的第五行和第六行干了什么，直接看第七行，count += len不是一个原子操作。假设这个时候count值为10，len值为1，两个线程同时执行到了第七行，拿到的count值都是10，执行完加法运算后将结果赋值给count，所以两个线程执行完后count值为11，而不是12。这就是为什么测试代码输出的值要比10000小的原因。

### 2、为什么会抛出ArrayIndexOutOfBoundsException异常。

我们看回`AbstractStringBuilder`的`append()`方法源码的第五行，`ensureCapacityInternal()`方法是检查`StringBuilder`对象的原char数组的容量能不能盛下新的字符串，如果盛不下就调用`expandCapacity()`方法对char数组进行扩容。

```java
private void ensureCapacityInternal(int minimumCapacity) {  
        // overflow-conscious code  
    if (minimumCapacity - value.length > 0)  
        expandCapacity(minimumCapacity);  
}  
```

扩容的逻辑就是new一个新的char数组，新的char数组的容量是原来char数组的两倍再加2，再通过System.arryCopy()函数将原数组的内容复制到新数组，最后将指针指向新的char数组。

```java
void expandCapacity(int minimumCapacity) {  
    //计算新的容量  
    int newCapacity = value.length * 2 + 2;  
    //中间省略了一些检查逻辑  
    ...  
    value = Arrays.copyOf(value, newCapacity);  
}  
```

`Arrys.copyOf()`方法

```java
public static char[] copyOf(char[] original, int newLength) {  
    char[] copy = new char[newLength];  
    //拷贝数组  
    System.arraycopy(original, 0, copy, 0,  
                         Math.min(original.length, newLength));  
    return copy;  
}  
```

`AbstractStringBuilder`的`append()`方法源码的第六行，是将String对象里面char数组里面的内容拷贝到`StringBuilder`对象的char数组里面，代码如下：

```java
str.getChars(0, len, value, count);  
```

getChars()方法

```java
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {  
    //中间省略了一些检查  
    ...     
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);  
    }  
```

拷贝流程见下图

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueq8rlWFnejuWibbkDsLW8SfkgV2icp12NDicCaAd0xklug4S51nyQCLicn9Lo9KospQpaKTfxmgAzEmQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

假设现在有两个线程同时执行了`StringBuilder`的append()方法，两个线程都执行完了第五行的`ensureCapacityInternal()`方法，此刻count=5。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueq8rlWFnejuWibbkDsLW8SfpC8jY6vIae2mn71v3LgR2nriavOj2aH8mueIibv2pRN3DbZ5zz6MrzOA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个时候线程1的`CPU`时间片用完了，线程2继续执行。线程2执行完整个`append()`方法后count变成6了

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueq8rlWFnejuWibbkDsLW8Sfh5uSy474qdNlYw4GwotlRoAqgsPOgHAicYlLOPZLeXtdWvJtMUfp5VA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

线程1继续执行第六行的`str.getChars()`方法的时候拿到的count值就是6了，执行char数组拷贝的时候就会抛出`ArrayIndexOutOfBoundsException`异常。

## 总结

至此，`StringBuilder`为什么不安全已经分析完了。如果我们将测试代码的`StringBuilder`对象换成`StringBuffer`对象会输出什么呢？

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueq8rlWFnejuWibbkDsLW8Sf2YLLBcpiaQ6Zlssaiaic2bGiaStKmarXgacjg7mPUeOmUKluiascsxZ75EA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当然是输出`10000`啦！

那么`StringBuffer`用什么手段保证线程安全的？这个问题你点进`StringBuffer`的`append()`方法里面就知道了。