# MySQL基础架构与日志详解

一、MySQL基础架构

MySQL可以分为Server层和存储引擎层两部分

Server层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖MySQL的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等

存储引擎负责数据的存储和提取。其架构模式是插件式的，支持InnoDB、MyISAM、Memory等多个存储引擎。现在最常用的存储引擎是InnoDB，它从MySQL 5.5.5版本开始成为了默认存储引擎。可以通过在SQL语句中使用engin=memory来指定使用内存引擎执行

不同的存储引擎共用一个Server层

1、连接器
连接器负责跟客户端建立连接、获取权限、维持和管理连接。连接命令一般是：

mysql -h$ip -P$port -u$user -p
1
连接命令中的mysql是客户端工具，用来跟服务端建立连接。在完成TCP握手后，连接器就要开始认证身份

如果用户名或密码不对，就会收到一个"Access denied for user"的错误，然后客户端程序结束执行
如果用户名密码认证通过，连接器回到权限表里面查出你拥有的权限。之后，这个连接里面的权限判断逻辑，都将依赖于此时读到的权限
这就意味着，一个用户成功建立连接后，即使用管理员帐号对这个用户的权限做了修改，也不会影响已经存在连接的权限。修改完成后，只有再新建的连接才会使用新的权限设置

连接完成后，如果你没有后续的动作，这个连接就处于空闲状态，可以在show processlist命令中看到它

Command为Sleep表示此连接是一个空闲连接

客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数wait_timeout控制的。默认值是8小时

如果在连接被断开之后，客户端再次发送请求的话，就会收到一个错误提示：Lost connection to MySQL server during query。这时候就需要重新连接，然后在执行请求了

数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个

建立连接的过程通常是比较复杂的，所以建议尽量使用长连接

但是全部使用长连接后，有些时候MySQL占用内存涨得特别快，这是因为MySQL在执行过程中临时使用的内存是管理在连接对象里面的。这些资源会在连接断开的时候才释放。所以如果长连接累计下来，可能导致内存占用太大，被系统强行杀掉（OOM），从现象看就是MySQL异常重启了

可以通过以下两种方案解决这个问题：

1.定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连

2.如果使用的是MySQL5.7或更新版本，可以在每次执行一个比较大的操作后，通过执行mysql_reset_connection来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态

2、查询缓存
建立连接完成后，可以执行select语句了。MySQL拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以key-value对的形式，被直接缓存在内存中。key是查询的语句，value是查询的结果。如果查询能够直接在这个缓存中找到key，那么这个value就会被直接返回给客户端

如果语句不在查询缓存中，就会继续后面的执行阶段。执行完成后，执行结果会被存入查询缓存中。如果查询命中缓存，MySQL不需要执行后面的复杂操作，就可以直接返回结果，这个效率很高

但是大多数情况下不建议使用查询缓存，因为查询缓存的失效非常频繁，只要对一个表的更新，这个表上所有的查询缓存都会被清空。对于更新压力大的数据库来说，查询缓存的命中率会非常低

可以将参数query_cache_type设置成DEMAND，这样对于默认的SQL语句都不使用查询缓存。而对于确定要是查询缓存的语句，可以用SQL_CACHE显示指定，如下面这条语句一样：

select SQL_CACHE * from T where ID=10；
1
MySQL8.0版本直接将查询缓存的整块功能删掉了

3、分析器
如果没有命中查询缓存，就要开始真正执行语句了。MySQL首先要对SQL语句做解析

分析器会先做词法分析。输入的是由多个字符串和空格组成的一条SQL语句，MySQL需要识别出里面的字符串分别是什么，代表什么

select * from T where ID=10；
1
MySQL从输入的select这个关键字识别出来，这是一个查询语句。它也要把字符串T识别成表名T，把字符串ID识别成列ID

做完了这些识别以后，就要做语法分析。根据词法分析的结果，语法分析器会根据语法规则，判断这个SQL语句是否满足MySQL语法。如果语法不对，就会收到"You have an error in your SQL syntax"的错误提示

4、优化器
经过了分析器，在开始执行之前，还要先经过优化器的处理

优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联的时候，决定各个表的连接顺序

5、执行器
优化器阶段完成后，这个语句的执行方案就确定下来了，然后进入执行器阶段，开始执行语句

开始执行的时候，要先判断一下你对这个表T有没有执行查询的权限，如果没有，就会返回没有权限的错误，如下所示

mysql> select * from T where ID=10;
ERROR 1142 (42000): SELECT command denied to user 'b'@'localhost' for table 'T'
1
2
如果有权限，就打开表继续执行。打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口

比如在表T中，ID字段没有索引，那么执行器的执行流程是这样的：

1.调用InnoDB引擎接口取这个表的第一行，判断ID值是不是10，如果不是则跳过，如果是则将这个行存在结果集中

2.调用引擎接口取下一行，重复相同的判断逻辑，直到取到这个表的最后一行

3.执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端

在数据库的慢查询日志中看到一个rows_examined的字段，表示这个语句执行过程扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的

在有些场景下，执行器调用一次，在引起内部则扫描了多行，因此引擎扫描行数跟rows_examined并不是完全相同的

二、日志系统
表T的创建语句如下，这个表有一个主键ID和一个整型字段c：

create table T(ID int primary key, c int);
1
如果要将ID=2这一行的值加1，SQL语句如下：

update T set c=c+1 where ID=2;
1
1、redo log（重做日志）
在MySQL中，如果每次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本、查找成本都很高。MySQL里常说的WAL技术，全称是Write-Ahead Logging，它的关键点就是先写日志，再写磁盘

当有一条记录需要更新的时候，InnoDB引擎就会把记录写到redo log里面，并更新buffer pool的page，这个时候更新就算完成了

buffer pool是物理页的缓存，对InnoDB的任何修改操作都会首先在buffer pool的page上进行，然后这样的页面将被标记为脏页并被放到专门的flush list上，后续将由专门的刷脏线程阶段性的将这些页面写入磁盘

InnoDB的redo log是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB，从头开始写，写到末尾就又回到开头循环写

write pos是当前记录的位置，一边写一边后移，写到第3号文件末尾后就回到0号文件开头。check point是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件

write pos和check point之间空着的部分，可以用来记录新的操作。如果write pos追上check point，这时候不能再执行新的更新，需要停下来擦掉一些记录，把check point推进一下

有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe

2、binlog（归档日志）
MySQL整体来看就有两块：一块是Server层，主要做的是MySQL功能层面的事情；还有一块是引擎层，负责存储相关的具体事宜。redo log是InnoDB引擎特有的日志，而Server层也有自己的日志，称为binlog

为什么会有两份日志？

因为最开始MySQL里并没有InnoDB引擎。MySQL自带的引擎是MyISAM，但是MyISAM没有crash-safe的能力，binlog日志只能用于归档。而InnoDB是以插件形式引入MySQL的，既然只依靠binlog是没有crash-safe能力的，所以InnoDB使用redo log来实现crash-safe能力

binlog的日志格式：

binlog的格式有三种：STATEMENT，ROW，MIXED

1）、STATEMENT模式

binlog里面记录的就是SQL语句的原文。优点是并不需要记录每一行的数据变化，减少了binlog日志量，节约IO，提高性能。缺点是在某些情况下会导致master-slave中的数据不一致(如sleep()函数， last_insert_id()，以及user-defined functions(udf)等会出现问题)

2）、ROW模式

不记录每条SQL语句的上下文信息，仅需记录哪条数据被修改了，修改成什么样了。而且不会出现某些特定情况下的存储过程或function或trigger的调用和触发无法被正确复制的问题。缺点是会产生大量的日志，尤其是alter table的时候会让日志暴涨

3）、MIXED模式

以上两种模式的混合使用，一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog，MySQL会根据执行的SQL语句选择日志保存方式

3、redo log和binlog日志的不同
1.redo log是InnoDB引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用

2.redo log是物理日志，记录的是在某个数据也上做了什么修改；binlog是逻辑日志，记录的是这个语句的原始逻辑，比如给ID=2这一行的c字段加1

3.redo log是循环写的，空间固定会用完；binlog是可以追加写入的，binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志

4、两阶段提交
执行器和InnoDB引擎在执行这个update语句时的内部流程：

1.执行器先找到引擎取ID=2这一行。ID是主键，引擎直接用树搜索找到这一行。如果ID=2这一行所在的数据也本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回

2.执行器拿到引擎给的行数据，把这个值加上1，得到新的一行数据，再调用引擎接口写入这行新数据

3.引擎将这行新数据更新到内存中，同时将这个更新操作记录到redo log里面，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务

4.执行器生成这个操作的binlog，并把binlog写入磁盘

5.执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成提交状态，更新完成

update语句的执行流程图如下，图中浅色框表示在InnoDB内部执行的，深色框表示是在执行器中执行的

将redo log的写入拆成了两个步骤：prepare和commit，这就是两阶段提交

由于redo log和binlog是两个独立的逻辑，如果不用两阶段提交，要么就是先写完redo log再写binlog，或者先写完binlog再写redo log

1.先写完redo log再写binlog。如果在redo log写完，binlog还没有写完的时候，MySQL进程异常重启。由于redo log写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行c的值是1。但是由于binlog还没写完就crash了，这时候binlog里面就没有记录这个语句，binlog中记录的这一行c的值为0

2.先写binlog后写redo log。如果在binlog写完之后crash，由于redo log还没写，崩溃恢复以后这个事务无效，所以这一行的c的值是0。但是binlog里面已经记录了把c从0改成1这个日志。所以，在之后binlog来恢复的时候就多了一个事务出来，恢复出来的这一行c的值就是1

如果不使用两阶段提交，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。redo log和binlog都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致

redo log用于保证crash-safe能力。innodb_flush_log_at_trx_commit这个参数设置成1的时候，表示每次事务的redo log都直接持久化到磁盘，这样可以保证MySQL异常重启之后数据不丢失

sync_binlog这个参数设置成1的时候，表示每次事务的binlog都持久化到磁盘，这样可以保证MySQL异常重启之后binlog不丢失

三、MySQL刷脏页
1、刷脏页的场景
当内存数据页跟磁盘数据页不一致的时候，我们称这个内存页为脏页。内存数据写入到磁盘后，内存和磁盘行的数据页的内容就一致了，称为干净页

第一种场景是，InnoDB的redo log写满了，这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写

checkpoint位置从CP推进到CP’，就需要将两个点之间的日志对应的所有脏页都flush到磁盘上。之后，上图中从write pos到CP’之间就是可以再写入的redo log的区域

第二种场景是，系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是脏页，就要先将脏页写到磁盘

这时候不能直接把内存淘汰掉，下次需要请求的时候，从磁盘读入数据页，然后拿redo log出来应用不就行了？

这里是从性能考虑的。如果刷脏页一定会写盘，就保证了每个数据页有两种状态：一种是内存里存在，内存里就肯定是正确的结果，直接返回；另一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样的效率最高

第三种场景是，MySQL认为系统空闲的时候刷脏页，当然在系统忙的时候也要找时间刷一点脏页
第四种场景是，MySQL正常关闭的时候会把内存的脏页都flush到磁盘上，这样下次MySQL启动的时候，就可以直接从磁盘上读数据，启动速度会很快
redo log写满了，要flush脏页，出现这种情况的时候，整个系统就不能再接受更新了，所有的更新都必须堵住

内存不够用了，要先将脏页写到磁盘，这种情况是常态。InnoDB用缓冲池管理内存，缓冲池中的内存页有三种状态：

第一种是还没有使用的
第二种是使用了并且是干净页
第三种是使用了并且是脏页
InnoDB的策略是尽量使用内存，因此对于一个长时间运行的库来说，未被使用的页面很少

当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页，即必须将脏页先刷到磁盘，变成干净页后才能复用

刷页虽然是常态，但是出现以下两种情况，都是会明显影响性能的：

一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长
日志写满，更新全部堵住，写性能跌为0，这种情况对敏感业务来说，是不能接受的
2、InnoDB刷脏页的控制策略
首先，要正确地告诉InnoDB所在主机的IO能力，这样InnoDB才能知道需要全力刷脏页的时候，可以刷多快。参数为innodb_io_capacity，建议设置成磁盘的IOPS

InnoDB的刷盘速度就是考虑脏页比例和redo log写盘速度。参数innodb_max_dirty_pages_pct是脏页比例上限，默认值是75%。脏页比例是通过Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total得到的，SQL语句如下：

mysql>  select VARIABLE_VALUE into @a from performance_schema.global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from performance_schema.global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
1
2
3
四、日志相关问题

问题一：在两阶段提交的不同时刻，MySQL异常重启会出现什么现象

如果在图中时刻A的地方，也就是写入redo log处于prepare阶段之后、写binlog之前，发生了崩溃，由于此时binlog还没写，redo log也还没提交，所以崩溃恢复的时候，这个事务会回滚。这时候，binlog还没写，所以也不会传到备库

如果在图中时刻B的地方，也就是binlog写完，redo log还没commit前发生崩溃，那崩溃恢复的时候MySQL怎么处理？

崩溃恢复时的判断规则：

1）如果redo log里面的事务是完整的，也就是已经有了commit标识，则直接提交

2）如果redo log里面的事务只有完整的prepare，则判断对应的事务binlog是否存在并完整

a.如果完整，则提交事务

b.否则，回滚事务

时刻B发生崩溃对应的就是2(a)的情况，崩溃恢复过程中事务会被提交

问题二：MySQL怎么知道binlog是完整的？

一个事务的binlog是有完整格式的：

statement格式的binlog，最后会有COMMIT
row格式的binlog，最后会有一个XID event
问题三：redo log和binlog是怎么关联起来的？

它们有一个共同的数据字段，叫XID。崩溃恢复的时候，会按顺序扫描redo log：

如果碰到既有prepare、又有commit的redo log，就直接提交
如果碰到只有prepare、而没有commit的redo log，就拿着XID去binlog找对应的事务
问题四：redo log一般设置多大？

如果是现在常见的几个TB的磁盘的话，redo log设置为4个文件、每个文件1GB

问题五：正常运行中的实例，数据写入后的最终落盘，是从redo log更新过来的还是从buffer pool更新过来的呢？

redo log并没有记录数据页的完整数据，所以它并没有能力自己去更新磁盘数据页，也就不存在数据最终落盘是由redo log更新过去的情况

1.如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘。这个过程，甚至与redo log毫无关系

2.在崩溃恢复场景中，InnoDB如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它对到内存，然后让redo log更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态

问题六：redo log buffer是什么？是先修改内存，还是先写redo log文件？

在一个事务的更新过程中，日志是要写多次的。比如下面这个事务：

begin;
insert into t1 ...
insert into t2 ...
commit;
1
2
3
4
这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没commit的时候就直接写到redo log文件里

所以，redo log buffer就是一块内存，用来先存redo日志的。也就是说，在执行第一个insert的时候，数据的内存被修改了，redo log buffer也写入了日志。但是，真正把日志写到redo log文件，是在执行commit语句的时候做的

五、MySQL是怎么保证数据不丢的？
只要redo log和binlog保证持久化到磁盘，就能确保MySQL异常重启后，数据可以恢复

1、binlog的写入机制
事务执行过程中，先把日志写到binlog cache，事务提交的时候，再把binlog cache写到binlog文件中。一个事务的binlog是不能被拆开的，因此不论这个事务多大，也要确保一次性写入

系统给binlog cache分配了一片内存，每个线程一个，参数binlog_cache_size用于控制单个线程内binlog cache所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘

事务提交的时候，执行器把binlog cache里的完整事务写入到binlog中，并清空binlog cache

每个线程有自己binlog cache，但是共用一份binlog文件

图中的write，指的就是把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，所以速度比较快
图中的fsync，才是将数据持久化到磁盘的操作。一般情况下认为fsync才占磁盘的IOPS
write和fsync的时机，是由参数sync_binlog控制的：

sync_binlog=0的时候，表示每次提交事务都只write，不fsync
sync_binlog=1的时候，表示每次提交事务都会执行fsync
sync_binlog=N（N>1）的时候，表示每次提交事务都write，但累积N个事务后才fsync
因此，在出现IO瓶颈的场景中，将sync_binlog设置成一个比较大的值，可以提升性能，对应的风险是：如果主机发生异常重启，会丢失最近N个事务的binlog日志

2、redo log的写入机制
事务在执行过程中，生成的redo log是要先写到redo log buffer的。redo log buffer里面的内容不是每次生成后都要直接持久化到磁盘，也有可能在事务还没提交的时候，redo log buffer中的部分日志被持久化到磁盘

redo log可能存在三种状态，对应下图的三个颜色块

这三张状态分别是：

存在redo log buffer中，物理上是在MySQL进程内存中，就是图中红色的部分
写到磁盘，但是没有持久化，物理上是在文件系统的page cache里面，也就是图中黄色的部分
持久化到磁盘，对应的是hard disk，也就是图中的绿色部分
日志写到redo log buffer和write到page cache都是很快的，但是持久化到磁盘的速度就慢多了

为了控制redo log的写入策略，InnoDB提供了innodb_flush_log_at_trx_commit参数，它有三种可能取值：

设置为0的时候，表示每次事务提交时都只是把redo log留在redo log buffer中
设置为1的时候，表示每次事务提交时都将redo log直接持久化到磁盘
设置为2的时候，表示每次事务提交时都只是把redo log写到page cache
InnoDB有一个后台线程，每隔1秒，就会把redo log buffer中的日志，调用write写到文件系统的page cache，然后调用fsync持久化到磁盘。事务执行中间过程的redo log也是直接写在redo log buffer中的，这些redo log也会被后台线程一起持久化到磁盘。也就是说，一个没有提交的事务的redo log也是可能已经持久化到磁盘的

还有两种场景会让一个没有提交的事务的redo log写入到磁盘中

1.redo log buffer占用的空间即将达到innodb_log_buffer_size一半的时候，后台线程会主动写盘。由于事务并没有提交，所以这个写盘动作只是write，而没有调用fsync，也就是只留在文件系统的page cache

2.并行的事务提交的时候，顺带将这个事务的redo log buffer持久化到磁盘。假设一个事务A执行到一半，已经写了一些redo log到buffer中，这时候有另外一个线程的事务B提交，如果innodb_flush_log_at_trx_commit设置的是1，事务B要把redo log buffer里的日志全部持久化到磁盘。这时候，就会带上事务A在redo log buffer里的日志一起持久化到磁盘

两阶段提交，时序上redo log先prepare，再写binlog，最后再把redo log commit。如果把innodb_flush_log_at_trx_commit设置成1，那么redo log在prepare阶段就要持久化一次

MySQL的双1配置，指的就是sync_binlog和innodb_flush_log_at_trx_commit都设置成1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是redo log（prepare阶段），一次是binlog

3、组提交机制
日志逻辑序列号LSN是单调递增的，用来对应redo log的一个个写入点，每次写入长度为length的redo log，LSN的值就会加上length。LSN也会写到InnoDB的数据页中，来确保数据页不会被多次执行重复的redo log

上图是三个并发事务在prepare阶段，都写完redo log buffer，持久化到磁盘的过程，对应的LSN分别是50、120和160

1.trx1是第一个到达的，会被选为这组的leader

2.等trx1要开始写盘的时候，这个组里面已经有了三个事务，这时候LSN也变成了160

3.trx1去写盘的时候，带的就是LSN=160，因此等trx1返回时，所有LSN小于等于160的redo log，都已经被持久化到磁盘

4.这时候trx2和trx3就可以直接返回了

一个组提交里面，组员越多，节约磁盘IOPS的效果要好

为了让一次fsync带的组员更多，MySQL做了拖时间的优化

binlog也可以组提交了，在执行上图第4步把binlog fsync到磁盘时，如果有多个事务的binlog已经写完了，也是一起持久化的，这样也可以减少IOPS的消耗

如果想提升binlog组提交的效果，可以通过设置binlog_group_commit_sync_delay和binlog_group_commit_sync_no_delay_count两个参数来实现

1.binlog_group_commit_sync_delay参数表示延迟多少微妙后才调用fsync

2.binlog_group_commit_sync_no_delay_count参数表示积累多少次以后才调用fsync

这两个条件只要有一个满足就会调用fsync

WAL机制主要得益于两方面：

redo log和binlog都是顺序写的，磁盘的顺序写比随机写速度要快
组提交机制，可以大幅降低磁盘订单IOPS消耗
4、如果MySQL现在出现了性能瓶颈，而且瓶颈在IO上，可以通过哪些方法来提升性能
1.设置binlog_group_commit_sync_delay（延迟多少微妙后才调用fsync）和binlog_group_commit_sync_no_delay_count（积累多少次以后才调用fsync）参数，减少binlog的写盘次数。这个方法是基于额外的故意等待来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险

2.将sync_binlog设置为大于1的值（每次提交事务都write，但累积N个事务后才fsync）。这样做的风险是，主机掉电的时候会丢binlog日志

3.将innodb_flush_log_at_trx_commit设置为2（每次事务提交时都只是把redo log写到page cache）。这样做的风险是，主机掉电的时候会丢数据