MySQL 中的 INSERT 是怎么加锁的？

在之前的博客中，我写了一系列的文章，比较系统的学习了 MySQL 的事务、隔离级别、加锁流程以及死锁，我自认为对常见 SQL 语句的加锁原理已经掌握的足够了，但看到热心网友在评论中[提出的一个问题](https://www.aneasystone.com/archives/2017/11/solving-dead-locks-two.html#comment-803)，我还是彻底被问蒙了。他的问题是这样的：

> 加了插入意向锁后，插入数据之前，此时执行了 select...lock in share mode 语句（没有取到待插入的值），然后插入了数据，下一次再执行 select...lock in share mode（不会跟插入意向锁冲突），发现多了一条数据，于是又产生了幻读。会出现这种情况吗？

这个问题初看上去很简单，在 RR 隔离级别下，假设要插入的记录不存在，如果先执行 `select...lock in share mode` 语句，很显然会在记录间隙之间加上 GAP 锁，而 `insert` 语句首先会对记录加插入意向锁，插入意向锁和 GAP 锁冲突，所以不存在幻读； 如果先执行 `insert` 语句后执行 `select...lock in share mode` 语句，由于 `insert` 语句在插入记录之后，会对记录加 X 锁，它会阻止 `select...lock in share mode` 对记录加 S 锁，所以也不存在幻读。两种情况如下所示：

先执行 INSERT 后执行 SELECT：

![insert-before-select-locks.jpg](https://www.aneasystone.com/usr/uploads/2018/06/2809704037.jpg)

先执行 SELECT 后执行 INSERT：

![insert-after-select-locks.jpg](https://www.aneasystone.com/usr/uploads/2018/06/2323649047.jpg)

但是我们仔细想一想就会发现哪里有点不对劲，我们知道 `insert` 语句会先在插入间隙上加上插入意向锁，然后开始写数据，写完数据之后再对记录加上 X 记录锁（这里简化了，关于 insert 语句的加锁流程，可以参考我之前写的 [常见 SQL 语句的加锁分析](https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html)）。那么问题就来了，如果在 `insert` 语句加插入意向锁之后，写数据之前，执行了 `select...lock in share mode` 语句，这个时候 GAP 锁和插入意向锁是不冲突的，查询出来的记录数为 0，然后 `insert` 语句写数据，加 X 记录锁，因为记录锁和 GAP 锁也是不冲突的，所以 `insert` 成功插入了一条数据，这个时候如果事务提交，`select...lock in share mode` 语句再次执行查询出来的记录数就是 1，岂不是就出现了幻读？

整个流程如下所示（我们把 `insert` 语句的执行分成两个阶段，INSERT 1 加插入意向锁，还没写数据，INSERT 2 写数据，加记录锁）：

![insert-12-select-locks.jpg](https://www.aneasystone.com/usr/uploads/2018/06/128585819.jpg)

## 一、INSERT 加锁的困惑

在得出上面的结论时，我也感到很惊讶。按理是不可能出现这种情况的，只可能是我对这两个语句的加锁过程还没有想明白。于是我又去复习了一遍 MySQL 官方文档，[Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html) 这篇文档对各个语句的加锁有详细的描述，其中对 `insert` 的加锁过程是这样说的（这应该是网络上介绍 MySQL 加锁机制被引用最多的文档，估计也是被误解最多的文档）：

> INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.
>
> Prior to inserting the row, a type of gap lock called an insert intention gap lock is set. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6 each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.
>
> If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session already has an exclusive lock. This can occur if another session deletes the row.

这里讲到了 `insert` 会对插入的这条记录加排他记录锁，在加记录锁之前还会加一种 GAP 锁，叫做插入意向锁，如果出现唯一键冲突，还会加一个共享记录锁。这和我之前的理解是完全一样的，那么究竟是怎么回事呢？难道 MySQL 的 RR 真的会出现幻读现象？

在 Google 上搜索了很久，并没有找到 MySQL 幻读的问题，百思不得其解之际，遂决定从 MySQL 的源码中一探究竟。

## 二、编译 MySQL 源码

编译 MySQL 的源码非常简单，但是中间也有几个坑，如果能绕过这几个坑，在本地调试 MySQL 是一件很容易的事（当然能调试源码是一回事，能看懂源码又是另一回事了）。

我的环境是 Windows 10 x64，系统上安装了 Visual Studio 2012，如果你的开发环境和我不一样，编译步骤可能也会不同。

在开始之前，首先要从官网下载 MySQL 源码（[下载地址](https://dev.mysql.com/downloads/mysql/5.6.html#downloads)）：

![download-mysql-source.jpg](https://www.aneasystone.com/usr/uploads/2018/06/2298955155.jpg)

这里我选择的是 5.6.40 版本，操作系统下拉列表里选 Source Code，OS Version 选择 Windows（Architecture Independent），然后就可以下载打包好的 zip 源码了。

将源码解压缩到 `D:\mysql-5.6.40` 目录，在编译之前，还需要再安装几个必要软件：

- [CMake](https://cmake.org/download/)：CMake 本身并不是编译工具，它是通过编写一种平台无关的 CMakeList.txt 文件来定制编译流程的，然后再根据目标用户的平台进一步生成所需的本地化 Makefile 和工程文件，如 Unix 的 Makefile 或 Windows 的 Visual Studio 工程；
- [Bison](http://gnuwin32.sourceforge.net/packages/bison.htm)：MySQL 在执行 SQL 语句时，必然要对 SQL 语句进行解析，一般来说语法解析器会包含两个模块：词法分析和语法规则。词法分析和语法规则模块有两个较成熟的开源工具 Flex 和 Bison 分别用来解决这两个问题。MySQL 出于性能和灵活考虑，选择了自己完成词法解析部分，语法规则部分使用了 Bison，所以这里我们还要先安装 Bison。Bison 的默认安装路径为 `C:\Program Files\GnuWin32`，**但是千万不要这样，一定要记得选择一个不带空格的目录**，譬如 `C:\GnuWin32` 要不然在后面使用 Visual Studio 编译 MySQL 时会卡死；
- Visual Studio：没什么好说的，Windows 环境下估计没有比它更好的开发工具了吧。

安装好 CMake 和 Bison 之后，记得要把它们都加到 PATH 环境变量中。做好准备工作，我们就可以开始编译了，首先用 CMake 生成 Visual Studio 的工程文件：

cmake 的 `-G` 参数用于指定生成哪种类型的工程文件，这里是 Visual Studio 2012，可以直接输入 `cmake -G` 查看支持的工程类型。如果没问题，会在 project 目录下生成一堆文件，其中 MySQL.sln 就是我们要用的工程文件，使用 Visual Studio 打开它。

打开 MySQL.sln 文件，会在 Solution Explorer 看到 130 个项目，其中有一个叫 ALL_BUILD，这个时候如果直接编译，编译会失败，在这之前，我们还要对代码做点修改：

- 首先是 `sql\sql_locale.cc` 文件，看名字就知道这个文件用于国际化与本土化，这个文件里有各个国家的语言字符，但是这个文件却是 ANSI 编码，所以要将其改成 Unicode 编码；
- 打开 `sql\mysqld.cc` 文件的第 5239 行，将 `DBUG_ASSERT(0)` 改成 `DBUG_ASSERT(1)`，要不然调试时会触发断言；

现在我们可以编译整个工程了，选中 ALL_BUILD 项目，Build，然后静静的等待 5 到 10 分钟，如果出现了 *Build: 130 succeeded, 0 failed* 这样的提示，那么恭喜，你现在可以尽情的调试 MySQL 了。

我们将 mysqld 设置为 Startup Project，然后加个命令行参数 `--console`，这样可以在控制台里查看打印的调试信息：

![debugging-mysqld.jpg](E:\other\网络\assets\3793622845.jpg)

另外 `client\Debug\mysql.exe` 这个文件是对应的 MySQL 的客户端，可以直接双击运行，默认使用的用户为 **ODBC@localhost**，如果要以 root 用户登录，可以执行 `mysql.exe -u root`，不需要密码。

## 三、调试 INSERT 加锁流程

首先我们创建一个数据库 test，然后创建一个测试表 t，主键为 id，并插入测试数据：

然后我们开两个客户端会话，一个会话执行 `insert into t(id) value(30)`，另一个会话执行 `select * from t where id = 30 lock in share mode`。很显然，如果我们能在 `insert` 语句加插入意向锁之后写数据之前下个断点，再在另一个会话中执行 `select` 就可以模拟出这种场景了。

那么我们来找下 `insert` 语句是在哪加插入意向锁的。第一次看 MySQL 源码可能会有些不知所措，调着调着就会迷失在深深的调用层级中，我们看 `insert` 语句的调用堆栈，一开始时还比较容易理解，从 *mysql_parse* -> *mysql_execute_command* -> *mysql_insert* -> *write_record* -> *handler::ha_write_row* -> *innobase::write_row* -> *row_insert_for_mysql*，这里就进入 InnoDb 引擎了。

然后继续往下跟：

> row_ins_step*-> *row_ins* -> *row_ins_index_entry_step* -> *row_ins_index_entry* -> *row_ins_clust_index_entry* -> *row_ins_clust_index_entry_low* -> *btr_cur_optimistic_insert* -> *btr_cur_ins_lock_and_undo* -> *lock_rec_insert_check_and_lock*。

一路跟下来，都没有发现插入意向锁的踪迹，直到 `lock_rec_insert_check_and_lock` 这里：

```
if (lock_rec_other_has_conflicting(
        static_cast<enum lock_mode>(
            LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION),
        block, next_rec_heap_no, trx)) {
 
    /* Note that we may get DB_SUCCESS also here! */
    trx_mutex_enter(trx);
 
    err = lock_rec_enqueue_waiting(
        LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION,
        block, next_rec_heap_no, index, thr);
 
    trx_mutex_exit(trx);
} else {
    err = DB_SUCCESS;
}
```

这里是检查是否有和插入意向锁冲突的其他锁，如果有冲突，就将插入意向锁加到锁等待队列中。这很显然是先执行 select ... lock in share mode 语句再执行 insert 语句时的情景，插入意向锁和 GAP 冲突。但这不是我们要找的点，于是继续探索，但是可惜的是，直到 insert 执行结束，我都没有找到加插入意向锁的地方。

跟代码非常辛苦，我担心是因为我跟丢了某块的逻辑导致没看到加锁，于是我看了看加其他锁的地方，发现在 InnoDb 里行锁都是通过调 `lock_rec_add_to_queue`（没有锁冲突） 或者 `lock_rec_enqueue_waiting`（有锁冲突，需要等待其他事务释放锁） 来实现的，于是在这两个函数上下断点，执行一条 `insert` 语句，依然没有断下来，说明 `insert` 语句没有加任何锁！

到这里我突然想起之前做过的 `insert` 加锁的实验，执行 `insert` 之后，如果没有任何冲突，在 `show engine innodb status` 命令中是看不到任何锁的，**这是因为 insert 加的是隐式锁。什么是隐式锁？隐式锁的意思就是没有锁！**

所以，根本就不存在之前说的先加插入意向锁，再加排他记录锁的说法，在执行 `insert` 语句时，什么锁都不会加。这就有点意思了，如果 `insert` 什么锁都不加，那么如果其他事务执行 `select ... lock in share mode`，它是如何阻止其他事务加锁的呢？

答案就在于隐式锁的转换。

InnoDb 在插入记录时，是不加锁的。如果事务 A 插入记录且未提交，这时事务 B 尝试对这条记录加锁，事务 B 会先去判断记录上保存的事务 id 是否活跃，如果活跃的话，那么就帮助事务 A 去建立一个锁对象，然后自身进入等待事务 A 状态，这就是所谓的隐式锁转换为显式锁。

我们跟一下执行 `select` 时的流程，如果 `select` 需要加锁，则会走： *sel_set_rec_lock* -> *lock_clust_rec_read_check_and_lock* -> *lock_rec_convert_impl_to_expl*，`lock_rec_convert_impl_to_expl` 函数的核心代码如下：

```
impl_trx = trx_rw_is_active(trx_id, NULL);
 
if (impl_trx != NULL
    && !lock_rec_has_expl(LOCK_X | LOCK_REC_NOT_GAP, block,
              heap_no, impl_trx)) {
    ulint    type_mode = (LOCK_REC | LOCK_X
                 | LOCK_REC_NOT_GAP);
 
    lock_rec_add_to_queue(
        type_mode, block, heap_no, index,
        impl_trx, FALSE);
}
```

首先判断事务是否活跃，然后检查是否已存在排他记录锁，如果事务活跃且不存在锁，则为该事务加上排他记录锁。而本事务的锁是通过 `lock_rec_convert_impl_to_expl` 之后的 `lock_rec_lock` 函数来加的。

到这里，这个问题的脉络已经很清晰了：

1. 执行 `insert` 语句，判断是否有和插入意向锁冲突的锁，如果有，加插入意向锁，进入锁等待；如果没有，直接写数据，不加任何锁；
2. 执行 `select ... lock in share mode` 语句，判断记录上是否存在活跃的事务，如果存在，则为 `insert` 事务创建一个排他记录锁，并将自己加入到锁等待队列；

所以不存在网友所说的幻读问题。那么事情到此结束了么？并没有。

细心的你会发现，执行 `insert` 语句时，从判断是否有锁冲突，到写数据，这两个操作之间还是有时间差的，如果在这之间执行 `select ... lock in share mode` 语句，由于此时记录还不存在，所以也不存在活跃事务，不会触发隐式锁转换，这条语句会返回 0 条记录，并加上 GAP 锁；而 `insert` 语句继续写数据，不加任何锁，在 `insert` 事务提交之后，`select ... lock in share mode` 就能查到 1 条记录，这岂不是还有幻读问题吗？

为了彻底搞清楚这中间的细节，我们在 `lock_rec_insert_check_and_lock` 检查完锁冲突之后下个断点，然后在另一个事务中执行 `select ... lock in share mode`，如果它能成功返回 0 条记录，加上 GAP 锁，说明就存在幻读。不过事实上，这条 SQL 语句执行的时候卡住了，并不会返回 0 条记录。从 `show engine innodb status` 的 `TRANSACTIONS` 里我们看不到任何行锁冲突的信息，但是我们从 `RW-LATCH INFO` 中却可以看出一些端倪：

```
-------------
RW-LATCH INFO
-------------
RW-LOCK: 000002C97F62FC70
Locked: thread 10304 file D:\mysql-5.6.40\storage\innobase\btr\btr0cur.cc line 879  S-LOCK
RW-LOCK: 000002C976A3B998
Locked: thread 10304 file D:\mysql-5.6.40\storage\innobase\btr\btr0cur.cc line 256  S-LOCK
Locked: thread 10304 file d:\mysql-5.6.40\storage\innobase\include\btr0pcur.ic line 518  S-LOCK
Locked: thread 2820 file D:\mysql-5.6.40\storage\innobase\btr\btr0cur.cc line 256  S-LOCK
Locked: thread 2820 file D:\mysql-5.6.40\storage\innobase\row\row0ins.cc line 2339  S-LOCK
RW-LOCK: 000002C976A3B8A8  Waiters for the lock exist
Locked: thread 2820 file D:\mysql-5.6.40\storage\innobase\btr\btr0cur.cc line 256  X-LOCK
Total number of rw-locks 16434
OS WAIT ARRAY INFO: reservation count 10
--Thread 10304 has waited at btr0cur.cc line 256 for 26.00 seconds the semaphore:
S-lock on RW-latch at 000002C976A3B8A8 created in file buf0buf.cc line 1069
a writer (thread id 2820) has reserved it in mode  exclusive
number of readers 0, waiters flag 1, lock_word: 0
Last time read locked in file btr0cur.cc line 256
Last time write locked in file D:\mysql-5.6.40\storage\innobase\btr\btr0cur.cc line 256
OS WAIT ARRAY INFO: signal count 8
Mutex spin waits 44, rounds 336, OS waits 7
RW-shared spins 3, rounds 90, OS waits 3
RW-excl spins 0, rounds 0, OS waits 0
Spin rounds per wait: 7.64 mutex, 30.00 RW-shared, 0.00 RW-excl
```

这里列出了 3 个 `RW-LOCK：000002C97F62FC70、000002C976A3B998、000002C976A3B8A8`。其中可以看到最后一个 RW-LOCK 有其他线程在等待其释放（Waiters for the lock exist）。下面列出了所有等待该锁的线程，`Thread 10304 has waited at btr0cur.cc line 256 for 26.00 seconds the semaphore`，这里的 Thread 10304 就是我们正在执行 `select` 语句的线程，它卡在了 `btr0cur.cc` 的 256 行，我们查看 Thread 10304 的堆栈：

![mysql-cond-wait.jpg](E:\other\网络\assets\2216971953.jpg)

`btr0cur.cc` 的 256 行位于 `btr_cur_latch_leaves` 函数，如下所示，通过 `btr_block_get` 来加锁，看起来像是在访问 InnoDb B+ 树的叶子节点时卡住了：

```
case BTR_MODIFY_LEAF:
    mode = latch_mode == BTR_SEARCH_LEAF ? RW_S_LATCH : RW_X_LATCH;
    get_block = btr_block_get(
        space, zip_size, page_no, mode, cursor->index, mtr);
```

这里的 `latch_mode == BTR_SEARCH_LEAF`，所以加锁的 mode 为` RW_S_LATCH`。

这里要介绍一个新的概念，叫做 **Latch**，一般也把它翻译成 “锁”，但它和我们之前接触的行锁表锁（**Lock**）是有区别的。这是一种轻量级的锁，锁定时间一般非常短，它是用来保证并发线程可以安全的操作临界资源，通常没有死锁检测机制。Latch 可以分为两种：MUTEX（互斥量）和 RW-LOCK（读写锁），很显然，这里我们看到的是 RW-LOCK。

我们回溯一下 `select` 语句的调用堆栈：

> *ha_innobase::index_read* -> *row_search_for_mysql* -> *btr_pcur_open_at_index_side* -> *btr_cur_latch_leaves*，

从调用堆栈可以看出 `select ... lock in share mode` 语句在访问索引，那么为什么访问索引会被卡住呢？

接下来我们看看这个 RW-LOCK 是在哪里加上的？从日志里可以看到 `Locked: thread 2820 file D:\mysql-5.6.40\storage\innobase\btr\btr0cur.cc line 256 X-LOCK`，所以这个锁是线程 2820 加上的，加锁的位置也在 `btr0cur.cc` 的 256 行，查看函数引用，很快我们就查到这个锁是在执行 `insert` 时加上的，函数堆栈为：*row_ins_clust_index_entry_low* -> *btr_cur_search_to_nth_level* -> *btr_cur_latch_leaves*。

我们看这里的 `row_ins_clust_index_entry_low` 函数（无关代码已省略）：

```
UNIV_INTERN
dberr_t
row_ins_clust_index_entry_low(
/*==========================*/
    ulint        flags,    /*!< in: undo logging and locking flags */
    ulint        mode,    /*!< in: BTR_MODIFY_LEAF or BTR_MODIFY_TREE,
                depending on whether we wish optimistic or
                pessimistic descent down the index tree */
    dict_index_t*    index,    /*!< in: clustered index */
    ulint        n_uniq,    /*!< in: 0 or index->n_uniq */
    dtuple_t*    entry,    /*!< in/out: index entry to insert */
    ulint        n_ext,    /*!< in: number of externally stored columns */
    que_thr_t*    thr)    /*!< in: query thread */
{
    /* 开启一个 mini-transaction */
    mtr_start(&mtr);
     
    /* 调用 btr_cur_latch_leaves -> btr_block_get 加 RW_X_LATCH */
    btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_LE, mode,
                    &cursor, 0, __FILE__, __LINE__, &mtr);
     
    if (mode != BTR_MODIFY_TREE) {
        /* 不需要修改 BTR_TREE，乐观插入 */
        err = btr_cur_optimistic_insert(
            flags, &cursor, &offsets, &offsets_heap,
            entry, &insert_rec, &big_rec,
            n_ext, thr, &mtr);
    } else {
        /* 需要修改 BTR_TREE，先乐观插入，乐观插入失败则进行悲观插入 */
        err = btr_cur_optimistic_insert(
            flags, &cursor,
            &offsets, &offsets_heap,
            entry, &insert_rec, &big_rec,
            n_ext, thr, &mtr);
        if (err == DB_FAIL) {
            err = btr_cur_pessimistic_insert(
                flags, &cursor,
                &offsets, &offsets_heap,
                entry, &insert_rec, &big_rec,
                n_ext, thr, &mtr);
        }
    }
     
    /* 提交 mini-transaction */
    mtr_commit(&mtr);
}
```

这里是执行 `insert` 语句的关键，可以发现执行插入操作的前后分别有一行代码：`mtr_start()` 和 `mtr_commit()`。这被称为 **迷你事务（mini-transaction）**，既然叫做事务，那这个函数的操作肯定是原子性的，事实上确实如此，`insert` 会在检查锁冲突和写数据之前，会对记录所在的页加一个 RW-X-LATCH 锁，执行完写数据之后再释放该锁（实际上写数据的操作就是写 redo log（重做日志），将脏页加入 flush list，这个后面有时间再深入分析了）。这个锁的释放非常快，但是这个锁足以保证在插入数据的过程中其他事务无法访问记录所在的页。mini-transaction 也可以包含子事务，实际上在 `insert` 的执行过程中就会加多个 mini-transaction，这中间的过程可以参考这篇博客：[MySQL - InnoDB mini transation](http://xnerv.wang/what-innodb-mini-transation/)：

每个 mini-transaction 会遵守下面的几个规则：

- 修改一个页需要获得该页的 X-LATCH；
- 访问一个页需要获得该页的 S-LATCH 或 X-LATCH；
- 持有该页的 LATCH 直到修改或者访问该页的操作完成。

所以，最后的最后，真相只有一个：`insert` 和 `select ... lock in share mode` 不会发生幻读。整个流程如下：

1. 执行 `insert` 语句，对要操作的页加 RW-X-LATCH，然后判断是否有和插入意向锁冲突的锁，如果有，加插入意向锁，进入锁等待；如果没有，直接写数据，不加任何锁，结束后释放 RW-X-LATCH；
2. 执行 `select ... lock in share mode` 语句，对要操作的页加 RW-S-LATCH，如果页面上存在 RW-X-LATCH 会被阻塞，没有的话则判断记录上是否存在活跃的事务，如果存在，则为 `insert` 事务创建一个排他记录锁，并将自己加入到锁等待队列，最后也会释放 RW-S-LATCH；

asdasda 