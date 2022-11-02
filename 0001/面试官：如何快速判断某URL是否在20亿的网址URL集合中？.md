腾讯面试官：如何快速判断某URL是否在20亿的网址URL集合中？

## 前言

在面试中，很容易被问到这样一个问题：

> **一个网站有 20 亿 url 存在一个黑名单中，这个黑名单要怎么存？**
>
> **若此时随便输入一个 url，你如何快速判断该 url 是否在这个黑名单中？**
>
> **并且需在给定内存空间（比如：500M）内快速判断出**。

可能很多人首先想到的会是使用 `HashSet`，因为 `HashSet`基于 `HashMap`，理论上时间复杂度为：`O(1)`。达到了快速的目的，但是空间复杂度呢？URL字符串通过Hash得到一个Integer的值，Integer占4个字节，那20亿个URL理论上需要：`20亿*4/1024/1024/1024=7.45G`的内存，不满足空间复杂度的要求。

这里就引出本文要介绍的“布隆过滤器”。

## 何为布隆过滤器

百科上对布隆过滤器的介绍是这样的：

> 布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都比一般的算法要好的多，缺点是有一定的误识别率和删除困难。

是不是描述的比较抽象？那就直接了解其原理吧！

### 还是以上面的例子为例：

哈希算法得出的Integer的哈希值最大为：`Integer.MAX_VALUE=2147483647`，意思就是任何一个URL的哈希都会在0~2147483647之间。

那么可以定义一个2147483647长度的byte数组，用来存储集合所有可能的值。为了存储这个byte数组，系统只需要：`2147483647/8/1024/1024=256M`。

比如：某个URL（X）的哈希是2，那么落到这个byte数组在第二位上就是1，这个byte数组将是：000….00000010，重复的，将这20亿个数全部哈希并落到byte数组中。

### 判断逻辑：

如果byte数组上的第二位是1，那么这个URL（X）可能存在。为什么是可能？因为有可能其它URL因哈希碰撞哈希出来的也是2，这就是误判。

但是如果这个byte数组上的第二位是0，那么这个URL（X）就一定不存在集合中。

### 多次哈希：

为了减少因哈希碰撞导致的误判概率，可以对这个URL（X）用不同的哈希算法进行N次哈希，得出N个哈希值，落到这个byte数组上，如果这N个位置没有都为1，那么这个URL（X）就一定不存在集合中。 

## Guava的BloomFilter

Guava框架提供了布隆过滤器的具体实现：BloomFilter，使得开发不用再自己写一套算法的实现。

### 创建BloomFilter

BloomFilter提供了几个重载的静态 `create`方法来创建实例：

```java
public static <T> BloomFilter<T> create(Funnel<? super T> funnel, int expectedInsertions, double fpp);
public static <T> BloomFilter<T> create(Funnel<? super T> funnel, long expectedInsertions, double fpp);
public static <T> BloomFilter<T> create(Funnel<? super T> funnel, int expectedInsertions);
public static <T> BloomFilter<T> create(Funnel<? super T> funnel, long expectedInsertions);
```

最终还是调用：

```java
static <T> BloomFilter<T> create(Funnel<? super T> funnel, long expectedInsertions, double fpp, Strategy strategy);
// 参数含义：
// funnel 指定布隆过滤器中存的是什么类型的数据，有：IntegerFunnel，LongFunnel，StringCharsetFunnel。
// expectedInsertions 预期需要存储的数据量
// fpp 误判率，默认是0.03。
```

BloomFilter里byte数组的空间大小由 `expectedInsertions`， `fpp`参数决定，见方法： 

```java
static long optimalNumOfBits(long n, double p) {
    if (p == 0) {
        p = Double.MIN_VALUE;
    }
    return (long) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
}
```

真正的byte数组维护在类：`BitArray`中。

### **使用:**

最后通过：`put`和 `mightContain`方法，添加元素和判断元素是否存在。

## 算法特点

1、因使用哈希判断，时间效率很高。空间效率也是其一大优势。

2、有误判的可能，需针对具体场景使用。

3、因为无法分辨哈希碰撞，所以不是很好做删除操作。

## 类似问题

1、黑名单 

2、URL去重 

3、单词拼写检查 

4、Key-Value缓存系统的Key校验 

5、ID校验，比如订单系统查询某个订单ID是否存在，如果不存在就直接返回。



好了，今天就分享到这里，下载在遇到面试被问类似问题，你是否应该这样？

![1621148491276](E:\other\网络\assets\1621148491276.png)

















