如何在 Linux 下优雅的查看系统 CPU 信息

我们在进行机器学习的时候，肯定需要使用一个比较好的 GPU 显卡，其次就是一个性能强劲的 CPU 了。主频高的 CPU 在跑程序的时候，真的有时候比使用 GPU 都跑的快，所以如何查看自己机器的 CPU 就是必不可少的步骤了。我们常常选购笔记本或者服务器的时候，总是会看到 X 核 XG 这样的表示，今天我们就一起来了解下其中的一些常见术语吧！ 

### 查看 CPU 型号和频率 - model

通过 CPU 的型号，我们可以直观的分辨其好坏和优劣，而频率则反馈的是其性能如何。

```
# CPU型号
$ cat /proc/cpuinfo | grep "model name" | uniq
model name      : Intel(R) Xeon(R) CPU E5-2640 v4 @ 2.40GHz

# CPU频率
$ cat /proc/cpuinfo | grep "cpu MHz" | uniq
cpu MHz         : 1547.537
cpu MHz         : 1250.590
cpu MHz         : 2183.637
```

### 查看物理 CPU 个数 - chip

主板上实际插入的 CPU 数量，可以数不重复的 physical id 字段有几个，即可。

```
# 物理CPU数量
$ cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l
2
```

### 查看每个物理 CPU 中 core 的个数 - core - 核数

单块 CPU 上面能处理数据的芯片组的数量，如双核、四核等，成为 cpu cores。

```
# CPU核数
$ cat /proc/cpuinfo | grep "cpu cores" | uniq
cpu cores       : 10
```

### 查看逻辑 CPU 的个数 - processor

一般情况下，**逻辑 CPU = 物理 CPU 个数 × 每颗核数**，如果不相等的话，则表示服务器的 CPU 支持超线程技术。超线程技术(HTT)：简单来说，它可使处理器中的 1 颗内核如 2 颗内核那样在操作系统中发挥作用。这样一来，操作系统可使用的执行资源扩大了一倍，大幅提高了系统的整体性能，此时**逻辑 CPU = 物理 CPU 个数 × 每颗核数 × 2**。

```
# 逻辑CPU数
$ cat /proc/cpuinfo | grep "processor" | wc -l
40
```

### 查询系统 CPU 是否启用超线程 - HTT

```
# 查询方式
$ cat /proc/cpuinfo | grep -e "cpu cores"  -e "siblings" | sort | uniq
cpu cores       : 10
siblings        : 20
```

dasdasd

asad

