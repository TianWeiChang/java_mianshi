为什么delete表数据，磁盘空间却还是被占用

最近有个上位机获取下位机上报数据的项目，由于上报频率比较频繁且数据量大，导致数据增长过快，磁盘占用多。

为了节约成本，定期进行数据备份，并通过delete删除表记录。

# 明明已经执行了delete，可表文件的大小却没减小，令人费解

项目中使用Mysql作为数据库，对于表来说，一般为表结构和表数据。表结构占用空间都是比较小的，一般都是表数据占用的空间。

当我们使用 delete删除数据时，确实删除了表中的数据记录，但查看表文件大小却没什么变化。

# Mysql数据结构

凡是使用过mysql，对B+树肯定是有所耳闻的，MySQL InnoDB 中采用了 B+ 树作为存储数据的结构，也就是常说的索引组织表，并且数据时按照页来存储的。因此在删除数据时，会有两种情况：

- 删除数据页中的某些记录
- 删除整个数据页的内容

# 表文件大小未更改和mysql设计有关

比如想要删除 R4 这条记录：

InnoDB 直接将 R4 这条记录标记为删除，称为可复用的位置。如果之后要插入 ID 在 300 到 700 间的记录时，就会复用该位置。**由此可见，磁盘文件的大小并不会减少。**

通用删除整页数据也将记录标记删除，数据就复用用该位置，与删除默写记录不同的是，删除整页记录，当后来插入的数据不在原来的范围时，都可以复用位置，而如果只是删除默写记录，是需要插入数据符合删除记录位置的时候才能复用。

因此，无论是数据行的删除还是数据页的删除，都是将其标记为删除的状态，用于复用，所以文件并不会减小。

# 那怎么才能让表大小变小

DELETE只是将数据标识位删除，并没有整理数据文件，当插入新数据后，会再次使用这些被置为删除标识的记录空间，可以使用OPTIMIZE TABLE来回收未使用的空间，并整理数据文件的碎片。

```
OPTIMIZE TABLE 表名;
```

注意：OPTIMIZE TABLE只对MyISAM, BDB和InnoDB表起作用。

另外，也可以执行通过ALTER TABLE重建表

```
ALTER TABLE 表名 ENGINE=INNODB
```

**有人会问OPTIMIZE TABLE和ALTER TABLE有什么区别？**

alter table t engine = InnoDB（也就是recreate），而optimize table t 等于recreate+analyze

# Online DDL

最后，再说一下Online DDL，dba的日常工作肯定有一项是ddl变更，ddl变更会锁表，这个可以说是dba心中永远的痛，特别是执行ddl变更，导致库上大量线程处于“Waiting for meta data lock”状态的时候。因此在 5.6 版本后引入了 Online DDL。

Online DDL推出以前，执行ddl主要有两种方式copy方式和inplace方式，inplace方式又称为(fast index creation)。相对于copy方式，inplace方式不拷贝数据，因此较快。但是这种方式仅支持添加、删除索引两种方式，而且与copy方式一样需要全程锁表，实用性不是很强。Online方式与前两种方式相比，不仅可以读，还可以支持写操作。

执行online DDL语句的时候，使用ALGORITHM和LOCK关键字，这两个关键字在我们的DDL语句的最后面，用逗号隔开即可。示例如下：

> ALTER TABLE tbl_name ADD COLUMN col_name col_type, ALGORITHM=INPLACE, LOCK=NONE;

**ALGORITHM选项**

- INPLACE：替换：直接在原表上面执行DDL的操作。
- COPY：复制：使用一种临时表的方式，克隆出一个临时表，在临时表上执行DDL，然后再把数据导入到临时表中，在重命名等。这期间需要多出一倍的磁盘空间来支撑这样的 操作。执行期间，表不允许DML的操作。
- DEFAULT：默认方式，有MySQL自己选择，优先使用INPLACE的方式。

**LOCK选项**

- SHARE：共享锁，执行DDL的表可以读，但是不可以写。
- NONE：没有任何限制，执行DDL的表可读可写。
- EXCLUSIVE：排它锁，执行DDL的表不可以读，也不可以写。
- DEFAULT：默认值，也就是在DDL语句中不指定LOCK子句的时候使用的默认值。如果指定LOCK的值为DEFAULT，那就是交给MySQL子句去觉得锁还是不锁表。不建议使用，如果你确定你的DDL语句不会锁表，你可以不指定lock或者指定它的值为default，否则建议指定它的锁类型。

执行DDL操作时，ALGORITHM选项可以不指定，这时候MySQL按照INSTANT、INPLACE、COPY的顺序自动选择合适的模式。也可以指定ALGORITHM=DEFAULT，也是同样的效果。如果指定了ALGORITHM选项，但不支持的话，会直接报错。

OPTIMIZE TABLE 和 ALTER TABLE 表名 ENGINE=INNODB都支持Oline DDL，但依旧建议在业务访问量低的时候使用

# 总结

delete 删除数据时，其实对应的数据行并不是真正的删除，仅仅是将其标记成可复用的状态，所以表空间不会变小。

可以重建表的方式，快速将delete数据后的表变小（OPTIMIZE TABLE 或ALTER TABLE），在 5.6 版本后，创建表已经支持 Online 的操作，但最好是在业务低峰时使用