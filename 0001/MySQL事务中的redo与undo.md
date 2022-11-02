MySQL事务中的redo与undo

## **一 前言**

 众所周知**InnoDB 是一个事务性的存储引擎**，在上一小节我们提到事务有4种特性：原子性、一致性、隔离性和持久性，在事务中的操作，要么全部执行，要么全部不做，这就是事务的目的。

 那么事务的四种特性到底是基于什么机制实现呢？？？

- 1、事务的原子性、隔离性由锁机制实现，我们将在后续章节《数据库锁机制》中介绍
- 2、而事务的一致性和持久性由事务的 redo 日志和undo 日志来保证。
  redo log 是重做日志，提供再写入操作，实现事务的持久性；
  undo log 是回滚日志，提供回滚操作，保证事务的一致性。

本文将讨论关于事务中的redo和undo的几个问题：

- redo 日志与undo日志分别用于记录什么？
- redo 如何保证事务的持久性？
- undo 如何保证事务的一致性？
- undo log 是否是redo log的逆过程？

## **二 Redo log**

### **2.1 redo的作用**

记录的是尚未完成的操作，数据库崩溃则用其重做

### **2.2 redo的组成**

Redo log可以简单分为以下两个部分：

- 保存在内存中重做日志的缓冲 (redo log buffer),是易失的
- 保存在硬盘中重做日志文件 (redo log file)，是持久的

### **2.3 redo工作流程**

![img](https://pic3.zhimg.com/80/v2-5d8a64c0d163fedf4118b314145d0da2_720w.jpg)

**InnoDB 的更新操作采用的是 Write Ahead Log (预先日志持久化)策略，即先写日志，再写入磁盘。**

当一条记录更新时，redo流程大致如下

在内存更新数据后，会把更新后的记录写入到 redo log buffer 中。

```
第一步：InnoDB 会先把记录从硬盘读入内存
第二部：修改数据的内存拷贝
第三步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值
第四步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加写的方式
第五步：定期将内存中修改的数据刷新到磁盘中(注意注意注意，不是从redo log file刷入磁盘，而是从内存刷入磁盘，redo log file只在崩溃恢复数据时才用)，如果数据库崩溃，则依据redo log buffer、redo log file进行重做，恢复数据,这才是redo log file的价值所在
```

### **2.4 Write Ahead Log**

redo是如何保证事务的持久性的呢？？？

答案是**Force Log at Commit 机制**，即当事务commit提交时，innodb引擎先将 redo log buffer 写入到 redo log file 进行持久化，待事务的commit操作完成时才算完成。这种做法也被称为 **Write-Ahead Log(预先日志持久化)**，在持久化一个数据页之前，先将内存中相应的日志页持久化。

问题1：为何不直接将修改的数据写入磁盘，而是要write ahead log呢？

```
答案：用于崩溃恢复
详解：
undo日志是对原始数据的备份
redo日志是对原始数据的修改

原始数据的按照既定的数据结构存放在磁盘上，写入磁盘是要耗费巨大成本的，而写入redo相对容易一些，因为redo里毕竟只需要考虑存放改动的数据即可，所以内存数据写写入redo log file，然后内存数据才能写入磁盘，如此，在内存数据再写入磁盘时因为某种原因比如断电崩溃，那么还可以依据redo log file恢复数据，如下图所示。
```

![img](https://pic4.zhimg.com/80/v2-2fcec4fa5bfaf02ca31ce85ed7318a73_720w.jpg)

问题2：如何保证每次修改的数据都能写入redo log file呢？

```
# 储备知识1
O_DIRECT选项是在Linux系统中的选项，使用该选项后，对文件进行直接IO操作，不经过文件系统缓存，直接写入磁盘

# 储备知识2
redo log又称之为重做日志，因重做日志打开并没有O_DIRECT选项，所以重做日志先写入到文件系统缓存，然后才会刷入硬盘，即
Redo log buffer--->os cache(文件系统缓存)--->redo log file

如果在刷入redo log file前断电，则会丢失文件系统缓存中数据，数据未写入redo log file，
因为由内存写入redo log file在前，而由内存写入磁盘在后，所以redo log file写入失败，则数据丢失

# 那如何保证每次的修改都记入日志文件redo log file呢？？？
答案是fsync操作
在每次将redo buffer写入os cache文件系统缓存后，InnoDB存储引擎都需要调用一次 fsync操作,保证立即由os cache文件系统缓存写入redo log file

fsync是一种系统调用操作，其fsync的效率取决于磁盘的性能，因此磁盘的性能也影响了事务提交的性能，也就是数据库的性能。
```

问题3：脏页何时刷入磁盘呢？

```
# 储备知识：脏页
Buffer Pool 中更新的数据未刷新到磁盘中，该内存页我们称之为脏页。最终脏页的数据会刷新到磁盘中，将磁盘中的数据覆盖，这个过程与 redo log 不一定有关系。

# 答案
redo log 日志满了的情况下，会主动触发脏页刷新到磁盘
```

问题4：脏页只在redo log满的情况下才会刷入磁盘吗？

```
答案：no，以下几种情况同样会触发脏页的刷新

- 1、系统内存不足时，需要将一部分数据页淘汰掉，如果淘汰的是脏页，需要先将脏页同步到磁盘；
- 2、MySQL 认为空闲的时间，这种情况没有性能问题；
- 3、MySQL 正常关闭之前，会把所有的脏页刷入到磁盘，这种情况也没有性能问题。
```

问题5：脏页刷入会带来性能问题吗？ [rml_read_more]：

```
在生产环境中，如果我们开启了慢 SQL 监控，你会发现偶尔会出现一些用时稍长的 SQL。**这是因为脏页在刷新到磁盘时可能会给数据库带来性能开销，**导致数据库操作抖动。
```

### **2.5 参数innodb_flush_log_at_trx_commit**

上面提到的**Force Log at Commit机制**就是靠InnoDB存储引擎提供的参数`innodb_flush_log_at_trx_commit`来控制的

该参数控制 commit提交事务 时，**如何将 redo log buffer 中的日志刷新到 redo log file 中。**

- 1、当设置参数为1时，（默认为1，建议），表示事务提交时必须调用一次 `fsync` 操作，最安全的配置，保障持久性
- 2、当设置参数为2时，则在事务提交时只做 **write** 操作，只保证将redo log buffer写到系统的页面缓存中，不进行fsync操作，因此如果MySQL数据库宕机时 不会丢失事务，但操作系统宕机则可能丢失事务
- 3、当设置参数为0时，表示事务提交时不进行写入redo log操作，这个操作仅在master thread 中完成，而在master thread中每1秒进行一次重做日志的fsync操作，因此实例 crash 最多丢失1秒钟内的事务。（master thread是负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性）

拓展阅读

```
我们需要注意的是 InnoDB 的 redo log 的大小是固定的，分别有多个日志文件采用循环方式组成一个循环闭环，当写到结尾时，会回到开头循环写日志。我们可以通过参数 innodb_log_files_in_group 和 innodb_log_file_size 配置日志文件数量和每个日志文件的大小。
```

## **三 Undo log**

### **3.1 作用**

undo即撤销还原。

用于记录更改前的一份copy，在操作出错时，可以用于回滚、撤销还原，只将数据库逻辑地恢复到原来的样子

undo日志记录了什么？

```
比如有两个用户访问数据库，当然并发罗。A是更改，B是查询。

--A更改还没有提交，B查询的话，数据肯定为历史数据，这个历史数据就是来源于UNDO段，

--A更改未提交，需要回滚rollback，回滚rollback的数据也来至于UNDO段。

结论：为了并发时读一致性成功，那么DML操作，肯定先写UNDO段。
```

### **3.2 undo的存储位置**

在InnoDB存储引擎中，undo存储在回滚段(Rollback Segment)中,每个回滚段记录了1024个undo log segment，而在每个undo log segment段中进行undo 页的申请，在5.6以前，Rollback Segment是在共享表空间里的，5.6.3之后，可通过 innodb_undo_tablespace设置undo存储的位置。

### **3.3 undo的类型**

在InnoDB存储引擎中，undo log分为：

- insert undo log
- update undo log

insert undo log是指在insert 操作中产生的undo log，因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。

而update undo log记录的是对delete 和update操作产生的undo log，该undo log可能需要提供MVCC机制，因此不能再事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

补充：purge线程两个主要作用是：清理undo页和清除page里面带有Delete_Bit标识的数据行。在InnoDB中，事务中的Delete操作实际上并不是真正的删除掉数据行，而是一种Delete Mark操作，在记录上标识Delete_Bit，而不删除记录。是一种"假删除",只是做了个标记，真正的删除工作需要后台purge线程去完成。

### **3.4 undo log 是否是redo log的逆过程？**

![img](https://pic3.zhimg.com/80/v2-5d8a64c0d163fedf4118b314145d0da2_720w.jpg)

undo log 是否是redo log的逆过程？其实从前文就可以得出答案了，undo log是逻辑日志，对事务回滚时，只是将数据库逻辑地恢复到原来的样子，而redo log是物理日志，记录的是数据页的物理变化，显然undo log不是redo log的逆过程。

### **3.5 redo & undo总结**

下面是redo log + undo log的简化过程，便于理解两种日志的过程：

```
假设有A、B两个数据，值分别为1,2.
1. 事务开始
2. 记录A=1到undo log
3. 修改A=3
4. 记录A=3到 redo log
5. 记录B=2到 undo log
6. 修改B=4
7. 记录B=4到redo log
8. 将redo log写入磁盘
9. 事务提交
```

实际上，在insert/update/delete操作中，redo和undo分别记录的内容都不一样，量也不一样。在InnoDB内存中，一般的顺序如下：

- 写undo的redo
- 写undo
- 修改数据页
- 写Redo

因为，数据在没有commit前，是随时从内存中写入到表数据块的，属于脏数据。 数据库崩溃后即使使用redo流程进行redo操作，但是脏数据还在，脏数据怎么处理，就只能靠undo流程，使用undo数据块的旧数据覆盖了。

undo与redo的联系：但是不管是脏的还是旧的，都在redo日志中复制了一份。

- 1.undo是一种“数据文件datafile”，具有表空间，当然具有块block;
- 2.redo是一种“文件file”，没有表空间。
- 3.数据库在DML事务时，先创建undo
- 4.读一致性与一致性(scn相同）的区别
- 5.undo与rollback的区别：在undo（撤销还原流程）中会使用rollback（回滚）这个动作