面试官：为什么要尽量避免使用 IN 和 NOT IN 呢？

## WHY？

IN 和 NOT IN 是比较常用的关键字，为什么要尽量避免呢？

### 1、效率低

项目中遇到这么个情况：

t1表 和 t2表  都是150w条数据，600M的样子，都不算大。

但是这样一句查询 ↓

```
select * from t1 where phone not in (select phone from t2)
```

直接就把我跑傻了。。。十几分钟，检查了一下  phone在两个表都建了索引，字段类型也是一样的。原来not in 是不能命中索引的。。。。

改成 NOT EXISTS 之后查询 20s ，效率真的差好多。

```
select * from t1 
where  not  EXISTS (select phone from t2  where t1.phone =t2.phone)
```

### 2、容易出现问题，或查询结果有误 （不能更严重的缺点）

以 IN 为例。建两个表：test1 和 test2

```
create table test1 (id1 int)
create table test2 (id2 int)

insert into test1 (id1) values (1),(2),(3)
insert into test2 (id2) values (1),(2)
```

我想要查询，在test2中存在的  test1中的id 。使用IN的一般写法是：

```
select id1 from test1 
where id1 in (select id2 from test2)
```

结果是：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDuMABibR1dg8KOqU4k4GKDWKBWrVh1Wub5NEcIx61WibFTrNSXH5jWmLlYwZmT9pib0IYcbaj69gTfg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

OK 木有问题！

但是如果我一时手滑，写成了：

```
select id1 from test1 
where id1 in (select id1 from test2)
```

不小心把id2写成id1了 ，会怎么样呢?

结果是：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDuMABibR1dg8KOqU4k4GKDWnv4KmS71pTG1S4sH1Mqtotf1dv17MtVVJTaNFNLVRia1bXXzDZ1T28A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

EXCUSE ME！为什么不报错？

单独查询 `select id1 from test2` 是一定会报错: 消息 207，级别 16，状态 1，第 11 行 列名 'id1' 无效。

然而使用了IN的子查询就是这么敷衍，直接查出 1 2 3

这仅仅是容易出错的情况，自己不写错还没啥事儿，下面来看一下 NOT IN 直接查出错误结果的情况：

给test2插入一个空值：

```
insert into test2 (id2) values (NULL)
```

我想要查询，在test2中不存在的  test1中的id 。

```
select id1 from test1 
where id1 not in (select id2 from test2)
```

结果是：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDuMABibR1dg8KOqU4k4GKDWVoVm1VMtpV2WkBvEoBclfl3SkUxjtLgWCFbCvBSASOiccDGLiaeAyutQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

空白！显然这个结果不是我们想要的。我们想要3。为什么会这样呢？

原因是：NULL不等于任何非空的值啊！如果id2只有1和2， 那么3<>1 且 3<>2 所以3输出了，但是 id2包含空值，那么 3也不等于NULL 所以它不会输出。

> 跑题一句：建表的时候最好不要允许含空值，否则问题多多。

## HOW？

### 1、用 EXISTS 或 NOT EXISTS 代替

```
select *  from test1 
   where EXISTS (select * from test2  where id2 = id1 )

select *  FROM test1  
 where NOT EXISTS (select * from test2  where id2 = id1 )
```

### 2、用JOIN 代替

```
 select id1 from test1 
   INNER JOIN test2 ON id2 = id1 
   
 select id1 from test1 
   LEFT JOIN test2 ON id2 = id1 
   where id2 IS NULL
```

妥妥的没有问题了！

PS：那我们死活都不能用 IN 和 NOT IN 了么？并没有，一位大神曾经说过，如果是确定且有限的集合时，可以使用。如 IN （0，1，2）。