面试官：聊聊事务隔离级别和MVCC吧！

## 前言

从我们开始连接 `Mysql server` 开始，也就开启了一个 `session` 。在每一个会话中我们可以发起事务，但是对于 `server` 来说，可能就是多个事务在进行，从我们的角度上看，事务在访问数据的时候，其他需要访问该数据的事务等待，该事务提交之后，其他事务就可以访问。但是对于 `server` 这是损耗性能的，所以又要做到 `隔离性`，又要性能，这个就要舍弃一些 `隔离性`。

## 隔离性问题

我们先来看看不串行执行，事务访问的数据会发生什么：

- `Dirty write`：一个事务修改了另一个`未提交事务`修改过的数据；【脏写】

- `Dirty read`：一个事务读到了另一个`未提交事务`修改过的数据；【脏读】

- `Non-Repeatable Read` ：一个事务可以马上读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值；【不可重复读】

  > ⚠️ 在 `SessionB` 中有几次隐式事务提交，而 `SessionA` 中都能查到最新的值

| sessionA                                                | sessionB                                                    |
| ------------------------------------------------------- | ----------------------------------------------------------- |
| `Begin;`                                                |                                                             |
| `select from trace where key = '20041';`【'key'】       |                                                             |
|                                                         | `uptate trace set typeof = 'buttonkey' where key = '20041'` |
| `select from trace where key = '20041';`【'buttonkey'】 |                                                             |
|                                                         | `uptate trace set typeof = 'mainkey' where key = '20041'`   |
| `select from trace where key = '20041';`【'mainly'】    |                                                             |

- `Phantom`：一个事务条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来；【幻读】

  | sessionA                                              | sessionB                                              |
  | ----------------------------------------------------- | ----------------------------------------------------- |
  | Begin;                                                |                                                       |
  | select keyv from trace where id > 0;【'20041'】       |                                                       |
  |                                                       | insert into trace(keyv, typeod) value('20036', 'key') |
  | select * from trace where id > 0;【'20041', '20036'】 |                                                       |

  > ⚠️注意这里是读到了之前没有读到的记录，而不是反而读不到之前的记录，那对于先前已经读到的记录，之后又读取不到这种情况？其实这相当于对每一条记录都发生了不可重复读的现象。幻读只是重点强调了读取到了之前读取没有获取到的记录。

## 标准的四种隔离级别

上面说的四种事务并发执行问题，从严重级别上：

> ⚠️ 脏写 > 脏读 > 不可重复读 > 幻读

而设计隔离级别，隔离级别越低，越严重的问题就越可能发生。下面4种隔离级别：

- `READ UNCOMMITTED`：未提交读；
- `READ COMMITTED`：已提交读；
- `REPEATABLE READ`：可重复读；
- `SERIALIZABLE`：可串行化。

下面列出各种隔离级别下可能发生的问题：

| 隔离级别           | 脏读 | 不可重复读 | 幻读 |
| ------------------ | ---- | ---------- | ---- |
| `READ UNCOMMITTED` | ✅    | ✅          | ✅    |
| `READ COMMITTED`   | ❌    | ✅          | ✅    |
| `REPEATABLE READ`  | ❌    | ❌          | ✅    |
| `SERIALIZABLE`     | ❌    | ❌          | ❌    |

> ⚠️ 脏写实在是太严重了，哪种隔离级别都不允许脏写发生。

## MVCC

`MVCC(Multi-Version Concurrency Control)`多版本并发控制，总体来说，`MVCC`使用了一种乐观锁的实现形式，借助`undo log`的可见性来控制数据库的并发操作。

在`mvcc`中有两种读：

- `快照读`：读取的是当前事务的可见版本。最简单的`select`操作；
- `当前读`：读取的是当前版本。显示带`lock`的读操作，`Insert/Update/Delete`操作

### 2.1 版本链

之前的 `undo log` 看到链表的连接，形式就像版本链可以进行数据追溯。而对于`InnoDB`的表，聚簇索引记录中都包含两个必要的隐藏列：

- `trx_id`：事务对某条聚簇索引记录产生改变时，会把`事务id`赋值在`trx_id`；
- `roll_pointer`：改动聚簇索引记录会把数据旧的版本写入`undo log`，`roll_pointer`相当一个指针指向这个`undo log`，从而形成一个可追溯的链表；

| trx389                                                    | trx399                                                    |
| --------------------------------------------------------- | --------------------------------------------------------- |
| `begin;`                                                  |                                                           |
|                                                           | `begin;`                                                  |
| `update trace set type = 'mainkey' where key = '20041'`   |                                                           |
| `update trace set type = 'buttonkey' where key = '20041'` |                                                           |
| `commit;`                                                 |                                                           |
|                                                           | `update trace set type = 'key' where key = '20041'`       |
|                                                           | `update trace set type = 'buttonkey' where key = '20041'` |
|                                                           | `commit;`                                                 |

> ⚠️ 上图为什么没有在事务之间对同一条记录进行操作呢？这不就是一个事务修改了一个未提交事务的数据 ===> 【脏写】。这个在任何一个隔离级别都是不允许的。而`InnoDB`会通过对`记录加锁`，另一个事务更新就需要等待之前的事务提交释放锁才能提交。

每对记录产生改动，就会生成一个`undo log`，然后把旧值放到这条`undo log`中，也就是这条记录一个旧版本。随着我们的修改继续，所有曾经的版本就会被`roll_pointer`串成一个链表，这个版本链的头节点就是当前记录最新的值。

![图片](https://mmbiz.qpic.cn/mmbiz/Ria7oAjGLicKzZ4yr1pwcuW0ZKeCKC2vWOs0azIZ62m3zRZLdVKTKk0zEm7lA0Kp3ks4PGohDqpTelCicr5L0hSRg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.2 ReadView

从名字上看：读视图。这个是`mysql`创建视图的必要条件。其实`readview`由三部分组成：

- `m_ids`：当前活跃的`事务id`列表。如：`[2,3]`
- `min_trx_id`：生成`readview`时当前活跃事务中`最小事务id`
- `max_trx_id`：生成`readview`时，即将分配给下一个`事务id`值

还有一个需要注意的就是：`create_trx_id` 生成该`readview`的`事务id`。

> ⚠️ 之前说过：只有在对表中的记录做改动时（`INSERT、DELETE、UPDATE`）才会为事务分配事务id。`Insert undo log`：当事务提交之后可以直接丢弃；`Update undo log`：长事务会产生很多老的视图导致undo log无法删除而占用大量存储空间。如果只是一个`只读事务`，则不会分配`trx_id`，故`事务id`都默认为0。

说完结构了，讲讲为什么需要这个吧：对应4种隔离级别，在读的时候能读到哪一步的记录是关键：

- `READ UNCOMMITTED`：可以读到未提交事务修改过的记录，也就是记录的最新情况是可以获取的；
- `SERIALIZABLE`：加锁来实现串行读；
- `READ COMMITTED`：必须保证读到已提交事务修改过的记录，也就是说如果另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录；
- `REPEATABLE READ`：同上；

所以关键就是可以在版本链上读到哪个记录，哪个事务修改的记录对于当前事务是可见的。

### 2.3 可见性

借用一下`kafka`的高水位的概念，一图入魂：

![图片](https://mmbiz.qpic.cn/mmbiz/Ria7oAjGLicKzZ4yr1pwcuW0ZKeCKC2vWOZmLXRvkr114fmVFe3yM50hURdjJtWkukQE5Nb1lFiadQxa9l87Eztfg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.4 整体分析

下图举个3个事务在线的🌰：

![图片](https://mmbiz.qpic.cn/mmbiz/Ria7oAjGLicKzZ4yr1pwcuW0ZKeCKC2vWOLpIbvXoMjx5ibhWylSMtfoyDiaI3Yn8iaE9CVLciaX9xNibI4RQGDNEREFQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

此时看`trx_id`300的两次`select`，两次产生的`readview`是不一致的。

这里就说出`READ COMMITTED`和`REPEATABLE READ`的区别：

> ⚠️在`MySQL`中，`READ COMMITTED`和`REPEATABLE READ`隔离级别的的一个非常大的区别就是它们生成ReadView的时机不同。

- `READ COMMITTED` —— 每次读取数据前都生成一个ReadView
- `REPEATABLE READ` —— 在第一次读取数据时生成一个ReadView

所以上图的所呈现的是在`READ COMMITTED`隔离级别下的情况。然后根据可见性分析，两次查询的寻链也就清晰

![图片](https://mmbiz.qpic.cn/mmbiz/Ria7oAjGLicKzZ4yr1pwcuW0ZKeCKC2vWOfv29BibQVyg5z4m5XjGugexUqoCsiaCKfbSrrXa9zQUrx2SNNz0XiarZQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那`REPEATABLE READ`？

从名字上就知道`可重复读`，再结合上面的生成`readview`的时机和次数。不言而喻，两次查询所得到的结果都是一样，而且以第一次为准，之后的结果都是重复第一次。

## 总结

所谓`MVCC`，指的就是在使用`READ COMMITTD`、`REPEATABLE READ`这两种隔离级别事务在执行普通的`SELECT`操作时访问记录的版本链，其中借助`undo log`，可见性算法，以及`readview`来完成整个过程。

所谓的`MVCC`只是在进行普通的`select`查询【也就是快照读】才生效，其他一些不普通的之后再讲。