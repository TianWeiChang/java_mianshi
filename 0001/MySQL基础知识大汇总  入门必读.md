## MySQL基础知识大汇总 | 入门必读

MySQL基础概念相关的名词还是挺多的，比如3大范式、4种隔离界别、ACID、DQL、DML、DDL等，还有redo、undo、binlog等。

本文就统一整理下MySQL常见的基础概念，方便小伙伴们翻。

### DQL

指`select`查询语句(`Data query language`)，基本结构是由 `select`子句，`from`子句，`where`。

常见使用方案：

```sql
-- 格式
SELECT selection_list /*要查询的列名称*/
  FROM table_list /*要查询的表名称*/
  WHERE condition /*行条件*/
  GROUP BY grouping_columns /*对结果分组*/
  HAVING condition /*分组后的行条件*/
  ORDER BY sorting_columns /*对结果排序*/
  LIMIT offset_start, row_count /*结果限定*/

-- 按条件查询指定列
SELECT t_man.Mname,t_man.Mage FROM t_man WHERE t_man.Mage > 30

-- --------------查询条件----------------------------

-- 1.逻辑运算符
-- NOT : 取反
WHERE NOT t_man.Mage > 30
-- AND : 逻辑与
WHERE t_man.Mage > 30 AND t_man.Mname LIKE '_'
-- OR : 逻辑或
WHERE t_man.Mage > 30 OR t_man.Mname LIKE '_'

-- 2.比较运算符
=、<>、!=、>、>=、!>、<、<=、!<

-- 3.LIKE，用于模糊查询
-- % : 后面可以跟零个或多个字符
-- _ : 匹配任意单个字符
-- [ ] : 查询一定范围内的单个字符，包括两端数据
WHERE t_man.Mname LIKE '[周李]%'
-- [^] [!]: 表示不在一定范围内的单个字符，包括两端数据

-- 4.BETWEEN
between xx and xx
WHERE t_man.Mage BETWEEN 30 AND 31 (等同于 t_man.Mage>=30 AND t_man.Mage<=31)
not between xx and xx

-- 5.is (not) null
-- 在 where 子句中，需要用 is (not) null 判断空值，不能使用 = 判断空值
WHERE t_man.Mage is not null

-- 6.in 多条件
WHERE t_man.Mage IN (30,31)

-- 7.ALL SOME ANY
-- Some 和 any 等效，all 是大于最大者，any 是小于最小者
WHERE t_man.Mage > ALL(SELECT t_man.Mage FROM t_man WHERE t_man.Mname LIKE '张%')

-- 8.exists 和 no exists
WHERE exists (select * from t_man where t_man.Mid = 8001)

-- 9.Group by 分组
SELECT AVG(t_man.Mage) FROM t_man GROUP BY t_man.Msex

-- 10.Having 分组后条件
SELECT AVG(t_man.Mage) AS mk,t_man.Msex FROM t_man GROUP BY t_man.Msex HAVING mk > 30

-- 11.ORDER BY 排序 ASC,DESC
SELECT * FROM t_man ORDER BY t_man.Mid ASC

-- 12.DISTINCT 去重
SELECT DISTINCT(t_man.Msex) FROM t_man

-- 23.LIMIT 分页（显示第一行数据）
SELECT * FROM t_man LIMIT 0,1


-- 交叉连接（Cross Join），没有链接条件的表查询会出现笛卡儿积
SELECT * FROM t_man,t_dept
SELECT * FROM t_man JOIN t_dept

-- 内连接（inner Join 或 Join），两表中都有才显示，即两表的交集
SELECT * FROM t_man JOIN t_dept ON t_man.Mid = t_dept.Mid

-- 左外连接（Left outer Join），以左边表为主，左表全部显示，没有对应的就显示空，即左并集
SELECT * FROM t_man LEFT JOIN t_dept ON t_man.Mid = t_dept.Mid

-- 右外连接（Right outer Join），与左外连接相反
SELECT * FROM t_man RIGHT JOIN t_dept ON t_man.Mid = t_dept.Mid

-- 全连接（Full outer Join），默认不支持，但也其他方式可以实现。
SELECT * FROM t_man LEFT JOIN t_dept ON t_man.Mid = t_dept.Mid
UNION
SELECT * FROM t_man RIGHT JOIN t_dept ON t_man.Mid = t_dept.Mid

-- UNION ALL 与 UNION 区别是允许重复
```



![img](https://img2020.cnblogs.com/blog/1595409/202007/1595409-20200714153011122-1966155227.png) 

### DML

DML是**数据操纵语言**（`Data Manipulation Language`）的简称，包括最常见的SQL语句，例如`SELECT`，`INSERT`，`UPDATE`，`DELETE`等，它用于**存储**，**修改**，**检索**和**删除**数据库中的数据。 

### DCL

DCL是**数据控制语言**（`Data Control Language`）的简称，它包含诸如`GRANT`之类的命令，并且主要涉及数据库系统的权限，权限和其他控件。

- `GRANT` ：允许用户访问数据库的权限
- `REVOKE`：撤消用户使用GRANT命令赋予的访问权限

### DDL

DDL是数据定义语言（`Data Definition Language`）的简称，它处理数据库`schemas`和描述数据应如何驻留在数据库中。

- CREATE：创建数据库及其对象（如表，索引，视图，存储过程，函数和触发器）
- ALTER：改变现有数据库的结构
- DROP：从数据库中删除对象
- TRUNCATE：从表中删除所有记录，包括为记录分配的所有空间都将被删除
- COMMENT：添加注释
- RENAME：重命名对象

常用命令如下：

```sql
# 建表
CREATE TABLE sicimike  (
  id int(4) primary key auto_increment COMMENT '主键ID',
  name varchar(10) unique,
  age int(3) default 0,
  identity_card varchar(18)
  # PRIMARY KEY (id) // 也可以通过这种方式设置主键
  # UNIQUE KEY (name) // 也可以通过这种方式设置唯一键
  # key/index (identity_card, col1...) // 也可以通过这种方式创建索引
) ENGINE = InnoDB;

# 设置主键
alter table sicimike add primary key(id);

# 删除主键
alter table sicimike drop primary key;

# 设置唯一键
alter table sicimike add unique key(column_name);

# 删除唯一键
alter table sicimike drop index column_name;

# 创建索引
alter table sicimike add [unique/fulltext/spatial] index/key index_name (identity_card[(len)] [asc/desc])[using btree/hash]
create [unique/fulltext/spatial] index index_name on sicimike(identity_card[(len)] [asc/desc])[using btree/hash]
example： alter table sicimike add index idx_na(name, age);

# 删除索引
alter table sicimike drop key/index identity_card;
drop index index_name on sicimike;

# 查看索引
show index from sicimike;

# 查看列
desc sicimike;

# 新增列
alter table sicimike add column column_name varchar(30);

# 删除列
alter table sicimike drop column column_name;

# 修改列名
alter table sicimike change column_name new_name varchar(30);

# 修改列属性
alter table sicimike modify column_name varchar(22);

# 查看建表信息
show create table sicimike;

# 添加表注释
alter table sicimike comment '表注释';

# 添加字段注释
alter table sicimike modify column column_name varchar(10) comment '姓名';
```

### TCL

TCL是**事务控制语言**（`Transaction Control Language`）的简称，用于处理数据库中的事务

- `COMMIT`：提交事务
- `ROLLBACK`：在发生任何错误的情况下回滚事务

## 三大范式

关于范式，其实在关系型数据库中一共有六种范式：

第一范式（1NF）、第二范式（2NF）、第三范式（3NF）、巴斯-科德范式（BCNF）、第四范式(4NF）和第五范式（5NF，又称完美范式）。满足最低要求的范式是第一范式（1NF）。 



- **第一范式：****每个字段都是原子的，也就说不可再分解**
- **第二范式：****有主键，非主键字段依赖主键字段**
- **第三范式：****非主键之间不能相互依赖**

注意，三大范式是数据表的建议设计原则，并不是非得完全按照这个来设计，**具体设计还要根据实际场景来分析**。任何给定的数据通常有多种表示方法，完全的范式话和反范式化，以及二者的折中。在范式化数据库中，任何数据都会出现且只出现一次，相反在反范式化中，数据是冗余的。

##  ACID

ACID是事务的4个特性，分别是原子性、一致性、隔离性和持久性。

- **A：****atomicity，原子性，**一个事务必须被视为一个不可分割的最小工作单元，整个事务中的所有操作要么全部提交成功，要么全部失败回滚
- **C：****consistency，一致性，**数据库总是从一个一致性的状态转换到另一个一致性的状态
- **I：****isolation，隔离性，**通常来说，一个事务所做的修改在最终提交以前，对其他事务是不可见的（隔离级别在非提交读时不满足）
- **D：**一旦事务提交，则其所做的修改就会永久保存到数据库中。

**隔离级别**

数据库事务的4种隔离级别：

- **未提交读：**一个事务可以读到另外一个事务未提交的数据
- **提交度：**一个事务更新数据过程中，如果事务还未提交，其他事务读不到该数据
- **可重复读：**该级别保证了在同一个事务中，多次读取同样记录的结果是一样的，解决了“提交读”中不可重复读的问题。但是理论上还是无法解决幻读问题（通过间隙锁可解决幻读问题）
- **串行化：**将所有事务都进行串行化处理，等级最高的隔离级别。

**幻读问题**

幻读就是当事务在读取某个范围数据时，另一个事务又在该范围插入了新的数据，当之前的事务再次读取该范围数据时，就会产生幻行。**产生幻读的原因是之前的事务在读取数据的范围没有增加范围锁**（range-locks），也就是读取时只是锁定的行级共享锁，没有锁定整个查询区间或者表。

**常见索引结构**

- **B+树索引：**B+ 树是关系型数据库中常见的索引类型。注意：B+树所以并不能找到一个给定键值的具体行，只能找到被查找数据行所在的页，然后数据库将页读入内存，在内存中进行查找，最后得到要查找的数据
- **哈希索引：**InnoDB支持的哈希索引是自适应的，不能人为干预在一张表中生成哈希索引，innodb会根据表的使用情况自动生成哈希索引
- **全文索引：**InnoDB支持全文索引，但是每张表只能有一个全文检索的索引，一般都是使用倒排索引技术来实现

**聚集索引&非聚集索引**

- **聚集索引就是主键索引，其叶子节点就是记录的数据（页）。**
- **非聚集索引也叫做辅助索引，其叶子结点记录的是主键值。**

以表t为例说明如下：

- 
- 
- 
- 
- 
- 

```
create table T (ID int primary key,k int NOT NULL DEFAULT 0,s varchar(16) NOT NULL DEFAULT '',index k(k)) engine=InnoDB;insert into T values(100,1, 'aa'),(200,2,'bb'),(300,3,'cc'),(500,5,'ee'),(600,6,'ff');
```

表T对应的主键索引和辅助索引如下：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/yAyQKzCbAHbpaibKjiaJa9Ss6c12Ml2xSALMA0kxrWIxiayu3bTAUEIcHaib0jb3YCOeFl3hZbPgCfh5LKLbiaO5dmg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**几个日志：**

- **redo log：**记录的是页的物理操作，InnoDB通过将事务操作先写redo log，而不是将数据页的更新写磁盘，相当于将磁盘随机写(data文件)变成了顺序写(redo log)，后续在MySQL"空闲"时再慢慢写磁盘，提高服务器性能

- **undo log：**undo log保存了事务发生之前的数据的版本，可用于回滚，同时可提供多版本并发控制读（MVCC），也就是非锁定读。undo log是逻辑日志，在执行undo时，仅仅是将数据逻辑上恢复至事务之前的状态，而不是从物理页上操作的，这一条不同于redo log。事务开始时将当前版本生成undo log，undo也会产生redo来保证undo log可靠性

- **binlog：**binlog是mysql层面的归档日志，可用于主从复制和数据库基于时间点的还原。binlog记录的是逻辑日志，记录的是DDL和DML操作日志，可以简单认为是执行过的事务中的更新sql语句

- 慢查询、错误日志等

**几个文件：**

- **.ibd文件和.ibdata文件：**.ibd文件和ibdata文件都是存放innodb数据的文件，之所有有2个，因为innodb支持配置来决定是使用共享表空间还是独享表空间。独享表空间使用".ibd"文件存储数据，并且每个表有一个.ibd文件；如果使用共享表空间，则会使用ibdata文件，所有表公用一个（或者配置多个）ibdata文件。

- **.ifm文件：**存放表相关的元数据信息。