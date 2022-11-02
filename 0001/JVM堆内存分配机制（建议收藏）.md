# JVM堆内存分配机制（建议收藏）

你好，我是田哥

《[JVM内存空间]( )》一文提到了，创建对象的时候，对象是在堆内存中创建的。但堆内存又分为新生代和老年代，新生代又细分Eden空间、From Survivor空间、To Survivor空间。

我们创建的对象到底在哪里？

 

## 对象优先在Eden分配

堆内存分为新生代和老年代，新生代是用于存放使用后准备被回收的对象，老年代是用于存放生命周期比较长的对象。

大部分我们创建的对象，都属于生命周期比较短的，所以会存放在新生代。新生代又细分Eden空间、From Survivor空间、To Survivor空间，我们创建的对象，对象优先在Eden分配。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFdhqg57tRXQgzqrwohTIn4zLAXQYxHjb9sqmyIU1BzbBqzccUxzhrzJA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

随着对象的创建，Eden剩余内存空间越来越少，就会触发`Minor GC`，于是Eden的存活对象会放入From Survivor空间。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFdUKucxQotc0mdXyA2zCEAs4x50N8PtHZiaLqibz4ad2pWj4oexZW4lskA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

`Minor GC`后，新对象依然会往Eden分配。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFd1oiba2R5LdcmZhiaqicIkahcAzzT5O0FUCq665RCicdDsHHibfIkjRTuZUQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Eden剩余内存空间越来越少，又会触发`Minor GC`，于是Eden和From Survivor的存活对象会放入To Survivor空间。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFdZAJ9f54zv6W8OfvKW7NtuVfqpfnBzMykiaDvyCsld8IpogrqiaqfF1zw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 大对象直接进入老年代

在上面的流程中，如果一个对象很大，一直在Survivor空间复制来复制去，那很费性能，所以这些大对象直接进入老年代。

可以用`XX:PretenureSizeThreshold`来设置这些大对象的阈值。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFd8XDfeSC7Eqg3YFR97VNn190bwoUl1HRgEwJUZ34s8bxa2gs8HIOUBg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 长期存活的对象将进入老年代

在上面的流程中，如果一个对象Hello_A，已经经历了15次`Minor GC`还存活在Survivor空间中，那他即将转移到老年代。这个15可以通过`-XX:MaxTenuringThreshold`来设置的，默认是15。

虚拟机为了给对象计算他到底经历了几次`Minor GC`，会给每个对象定义了一个对象年龄计数器。如果对象在Eden中经过第一次Minor GC后仍然存活，移动到Survivor空间年龄加1，在Survivor区中每经历过Minor GC后仍然存活年龄再加1。年龄到了15，就到了老年代。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFdSHFxz22aXtkV33iaSaXGqLY54iaXXy9YU57K7STib06gxavIibPL8QKTuA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 动态年龄判断

除了年龄达到MaxTenuringThreshold的值，还有另外一个方式进入老年代，那就是动态年龄判断：在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。

比如Survivor是100M，Hello1和Hello2都是3岁，且总和超过了50M，Hello3是4岁，这个时候，这三个对象都将到老年代。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFd9iazvFqqpAMtewMUmGSfeG0U1N0MPoFEqTkAXqVdIRYvl44cdGblibxg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 空间分配担保

上面的流程提过，存活的对象都会放入另外一个Survivor空间，如果这些存活的对象比Survivor空间还大呢？整个流程如下：

1. Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果大于，则发起Minor GC。
2. 如果小于，则看HandlePromotionFailure有没有设置，如果没有设置，就发起full gc。
3. 如果设置了HandlePromotionFailure，则看老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果小于，就发起full gc。
4. 如果大于，发起Minor GC。Minor GC后，看Survivor空间是否足够存放存活对象，如果不够，就放入老年代，如果够放，就直接存放Survivor空间。如果老年代都不够放存活对象，担保失败（Handle Promotion Failure），发起full gc。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PxMzT0Oibf4ghFum77IVictLuVLkY5IjFdCJeKdtia7NibNRSDTMcggvEwmCr1NgqUTACibuqjQiasb0TiaDYuZSznXTA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



 阿萨德大

有三个方面的原因：

- 在1.7版本里面，永久代内存是有上限的，虽然我们可以通过参数来设置，但是JVM加载的class总数、大小是很难确定的。

  所以很容易出现OOM问题。

  但是元空间是存储在本地内存里面，内存上限比较大，可以很好的避免这个问题。

- 永久代的对象是通过FullGC进行垃圾收集，也就是和老年代同时实现垃圾收集。

  替换成元空间以后，简化了Full GC。可以在不进行暂停的情况下并发地释放类数据，同时也提升了GC的性能

- Oracle要合并Hotspot和JRockit的代码，而JRockit没有永久代。