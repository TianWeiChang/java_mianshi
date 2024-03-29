MySQL的自增 ID 用完了，怎么办？

如果你用过或了解过MySQL，那你一定知道自增主键了。每个自增id都是定义了初始值，然后按照指定步长增长（默认步长是1）。虽然，自然数是没有上限的，但是我们在设计表结构的时候，通常都会指定字段长度，那么，这时候id就有上限了。既然有上限，就总有被用完的时候，如果id用完了，怎么办呢？今天就一起来学习下吧。 

## 自增id

说到自增id，相信你的第一反应一定是在设计表结构的时候自定义一个自增id字段，那么就有一个问题啦，在插入数据时有可能唯一主键冲、sql事务回滚、批量插入的时候，批量申请自增值等原因导致自增id是不连续的。

表定义的自增值达到上线后的逻辑是：再申请下一个id的时候，获取的是同一个值（最大值）。大家可以插入sql设置id是最大值，再insert一条不主动设置id的语句就可以验证这一结论啦。这个时候如果再插入就是报主键冲突咯～

这里提醒一下：232-1（4294967295）不是一个特别大的数，对于一个频繁插入删除数据的表来说，是可能会被用完的。因此在建表的时候你需要考察你的表是否有可能达到这个上限，如果有可能，就应该创建成 8 个字节的 bigint unsigned。

## InnoDB系统自增row_id

如果你创建的 InnoDB 表没有指定主键，那么 InnoDB 会给你创建一个不可见的，长度为 6 个字节的 row_id。InnoDB 维护了一个全局的 dict_sys.row_id 值，所有无主键的 InnoDB 表，每插入一行数据，都将当前的 dict_sys.row_id 值作为要插入数据的 row_id，然后把 dict_sys.row_id 的值加 1。

![图片](https://mmbiz.qpic.cn/mmbiz_png/bcPwoCALib9JdJYYm6JGRbj4vzVnJr7hzicPhvuHDZib2NPP2hd5WVbc5tKSGN7Vd6UHhZce8Uc3tJ0yaOe9GnbFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

实际上，在代码实现时 row_id 是一个长度为8字节的无符号长整型 (bigint unsigned)。但是，InnoDB 在设计时，给 row_id 留的只是 6 个字节的长度，这样写到数据表中时只放了最后 6 个字节，所以 row_id 能写到数据表中的值，就有两个特征：

### row_id 写入表中的值范围，是从 0 到 248-1；

### 当 dict_sys.row_id=2^48时，如果再有插入数据的行为要来申请 row_id，拿到以后再取最后 6 个字节的话就是 0。

虽然，2^48这个数字已经很大了，但是大家要知道 一个系统是可以跑很久的，那么还是可能达到上限的，这时候再申请就会覆盖原来的记录了。因此，尽量不要选择这种方式！

## Xid

MySQL中redo log 和 binlog 相配合的时候，它们有一个共同的字段叫作 Xid。它在 MySQL 中是用来对应事务的。

MySQL 内部维护了一个全局变量 global_query_id，每次执行语句的时候将它赋值给 Query_id，然后给这个变量加 1。如果当前语句是这个事务执行的第一条语句，那么 MySQL 还会同时把 Query_id 赋值给这个事务的 Xid。而 global_query_id 是一个纯内存变量，重启之后就清零了。所以在同一个数据库实例中，不同事务的 Xid 也是有可能相同的。

## Innodb trx_id

InnoDB 内部维护了一个 max_trx_id 全局变量，每次需要申请一个新的 trx_id 时，就获得 max_trx_id 的当前值，然后并将 max_trx_id 加 1。

InnoDB 数据可见性的核心思想是：每一行数据都记录了更新它的 trx_id，当一个事务读到一行数据的时候，判断这个数据是否可见的方法，就是通过事务的一致性视图与这行数据的 trx_id 做对比。但是这个过程有脏读存在，那么这个id就不会是原子性的，存在重复的可能性。

## thread_id

其实，线程 id 才是 MySQL 中最常见的一种自增 id。平时我们在查各种现场的时候，show processlist 里面的第一列，就是 thread_id。

thread_id 的逻辑很好理解：系统保存了一个全局变量 thread_id_counter，每新建一个连接，就将 thread_id_counter 赋值给这个新连接的线程变量。

thread_id_counter 定义的大小是 4 个字节，因此达到 232-1 后，它就会重置为 0，然后继续增加。结果跟row_id一样，就会覆盖原有记录了。

上面介绍了几种MySQL自身的一些自增id，其实，实际运用中，我们也可能会选择外部的自增主键，然后持久化到数据库，以此来代替数据库自身的自增id。下面来说说吧。

## Redis自增主键

其实外部自增主键的生成方式有很多，为什么我要介绍redis呢？因为我自己在实际应用中使用发现它的很多优点。

redis自身是原子性的，因此高并发也是线程安全的。假设主键字段长度20，我们以时间+自增数来构成主键，例如：8位日期+12自增数。那么，根据业务性质可以决定时间取年月日或者到毫秒级，那么在毫秒之间自增数的重复概率是极小极小的，基本的业务都能适用。

## 总结

上面介绍了好几种自增id，每种自增 id 有各自的应用场景，在达到上限后的表现也不同：

**1、** 表的自增 id 达到上限后，再申请时它的值就不会改变，进而导致继续插入数据时报主键冲突的错误
**2、** row_id 达到上限后，则会归 0 再重新递增，如果出现相同的 row_id，后写的数据会覆盖之前的数据
**3、** Xid 只需要不在同一个 binlog 文件中出现重复值即可。虽然理论上会出现重复值，但是概率极小，可以忽略不计
**4、** InnoDB 的 max_trx_id 递增值每次 MySQL 重启都会被保存起来，所以我们文章中提到的脏读的例子就是一个必现的 bug，好在留给我们的时间还很充裕
**5、** thread_id 是我们使用中最常见的，而且也是处理得最好的一个自增 id 逻辑了
**6、** redis外部自增，毫秒级别，理论上会出现重复值，但是概率极小，可以忽略不计
**7、** 其实，每种自增id都有各自的适用场景，大家在平时使用中可以根据具体场景再选择。

但是要未雨绸缪，因为系统的运行时间和数据的存储，这些都是要考虑在内的，综合考虑，选择一个在系统运行期间一定不会出现重复即刻。你学会了吗？