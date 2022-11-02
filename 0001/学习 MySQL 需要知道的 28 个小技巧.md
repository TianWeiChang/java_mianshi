学习 MySQL 需要知道的 28 个小技巧

随着信息技术的不断发展以及互联网行业的高速增长，作为开源数据库的MySQL得到了广泛的应用和发展。目前MySQL已成为关系型数据库领域中非常重要的一员。

无论是运维、开发、测试，还是架构师，数据库技术都是一个 **必备加薪神器**，那么，一直说学习数据库、学 **MySQL**，到底是要学习它的哪些东西呢？

## 如何快速掌握 MySQL?

**培养兴趣**

兴趣是最好的老师，不论学习什么知识，兴趣都可以极大地提高学习效率。不管学习 `MySQL5.7` 还是 `MySQL8.0` 都不例外！

**夯实 SQL 基础**

计算机领域的技术非常强调基础，刚开始学习可能还认识不到这一点。随着技术应用的深 入，只有有着扎实的基础功底，才能在技术的道路上走得更快、更远。对于 MySQL 的学习来说， **SQL 语句** 是其中最为基础的部分，很多操作都是通过 SQL 语句来实现的。所以在学习的过程中， 读者要多编写 SQL 语句，对于同一个功能，使用不同的实现语句来完成，从而深刻理解其不同之处。

**及时学习新知识**

正确、有效地利用搜索引擎，可以搜索到很多关于 MySQL 的相关知识。同时，参考别 人解决问题的思路，也可以吸取别人的经验，及时获取最新的技术资料。

**多实践操作**

数据库系统具有极强的操作性，需要多动手上机操作。在实际操作的过程中才能发现问题， 并思考解决问题的方法和思路，只有这样才能提高实战的操作能力。

下面分享学习 MySQL 的 28 个不得不知道的小技巧！

## 1、MySQL 中如何使用特殊字符？

诸如单引号 `'`，双引号 `"`，反斜线 `\` 等符号，这些符号在 MySQL 中不能直接输入使用，否则会产生意料之外的结果。

**举例：**

假设 Lucifer 表中需要存入一行记录，值为 `lucifer's dog`，其中的单引号 `'` 号，如果不做转义，则无法成功执行：

```
mysql> create table lucifer (id int,name char(100));
Query OK, 0 rows affected (0.02 sec)

mysql> insert into lucifer values (1,'lucifer's dog');
    '> 
    '> mysql> 

^C
mysql>
```

在 MySQL 中，这些特殊字符称为转义字符，在输入时需要以反斜线符号 `\` 开头，所以在使用单引号和双引号时应分别输入 `\'` 或者 `\"`，输入反斜线时应该输入 `\\`，其他特殊字符还有回车符 `\r`，换行符 `\n`，制表符 `\tab`，退格符 `\b` 等。

```
mysql> create table lucifer (id int,name char(100));
Query OK, 0 rows affected (0.03 sec)

mysql> insert into lucifer values (1,'lucifer\'s dog');
Query OK, 1 row affected (0.00 sec)

mysql> select * from lucifer;
+------+---------------+
| id   | name          |
+------+---------------+
|    1 | lucifer's dog |
+------+---------------+
1 row in set (0.00 sec)
mysql> 
```

**📢 注意：** 在向数据库中插入这些特殊字符时，一定要进行转义处理。

## 2、MySQL 中可以存储文件吗？

**答案当然是可以的！**

MySQL 中的 `BLOB` 和 `TEXT` 字段类型可以存储数据量较大的文件，可以使用这些数据类型 存储图像、声音或者是大容量的文本内容，例如网页或者文档。

```
mysql> create table view(id int unsigned NOT NULL AUTO_INCREMENT, catid int,title varchar(256),picture MEDIUMBLOB, content TEXT,PRIMARY KEY (id));
Query OK, 0 rows affected (0.03 sec)

mysql> show fields from view;
+---------+--------------+------+-----+---------+----------------+
| Field   | Type         | Null | Key | Default | Extra          |
+---------+--------------+------+-----+---------+----------------+
| id      | int unsigned | NO   | PRI | NULL    | auto_increment |
| catid   | int          | YES  |     | NULL    |                |
| title   | varchar(256) | YES  |     | NULL    |                |
| picture | mediumblob   | YES  |     | NULL    |                |
| content | text         | YES  |     | NULL    |                |
+---------+--------------+------+-----+---------+----------------+
5 rows in set (0.00 sec)

mysql> 
```

虽然使用 BLOB 或者 TEXT 可 以存储大容量的数据，但是对这些字段的处理会降低数据库的性能。

**📢 注意：** 如果并非必要，可以选择只储存文件的路径。

## 3、MySQL 中如何执行区分大小写的字符串比较？

MySQL 是 **不区分大小写** 的，因此字符串比较函数也不区分大小写。

```
mysql> select 'TRUE' from dual where 'DOG' = 'dog';
+------+
| TRUE |
+------+
| TRUE |
+------+
1 row in set (0.00 sec)
```

如果想执行区分大小写的比较，可以在字符串前面添加 BINARY 关键字。

```
mysql> select 'TRUE' from dual where BINARY'DOG' = 'dog';
Empty set (0.00 sec)

mysql> 
```

例如默认情况下，’DOG‘=’dog‘ 返回结果为 TRUE，如果使用 BINARY 关键字，BINARY’DOG’=‘dog’ 结果为 FALSE，在区分大小写的情况下，’DOG’  与 ’dog’ 并不相同。

## 4、如何从日期时间值中获取年、月、日等部分日期或时间值？

MySQL 中，日期时间值以字符串形式存储在数据表中，因此可以使用字符串函数分别截取日期时间值的不同部分。

```
mysql> create table lucifer(date date);
Query OK, 0 rows affected (0.04 sec)

mysql> show fields from lucifer;
+-------+------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+-------+------+------+-----+---------+-------+
| date  | date | YES  |     | NULL    |       |
+-------+------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql> insert into lucifer values (now());
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> select * from lucifer;
+------------+
| date       |
+------------+
| 2021-11-25 |
+------------+
1 row in set (0.00 sec)
```

例如某个名称为 date 的字段有值 `2021-11-25`，如果只需要获得年值，可以输入 `LEFT(date, 4)`，这样就获得了字符串左边开始长度为 4 的子字符串，即 `YEAR` 部分的值；

```
mysql> select LEFT(date, 4) from lucifer;
+---------------+
| LEFT(date, 4) |
+---------------+
| 2021          |
+---------------+
1 row in set (0.00 sec)
```

如果要获取月份值，可以输入 `MID(date,6,2)`，字符串第 6 个字符开始，长度为 2 的子字符串正好为 date 中的月份值。同理，读者可以根据其他日期和时间的位置，计算并获取相应的值。

```
mysql> select MID(date,6,2) from lucifer;
+---------------+
| MID(date,6,2) |
+---------------+
| 11            |
+---------------+
1 row in set (0.00 sec)
```

## 5、如何改变默认的字符集？

`CONVERT()` 函数改变指定字符串的默认字符集！

MySQL 的安装和配置过程中，其中的一个步骤是可以选择 MySQL 的默认字符集。但是，如果只改变字符集，没有必要把配置过程重新执行一遍，在这里，一个简单的方式是 `修改配置文件`。

读者可以在修改字符集时使用 `SHOW VARIABLES LIKE 'character_set_%';` 或者 `status` 命令查看当前字符集，以进行对比。

```
mysql> SHOW VARIABLES LIKE 'character_set_%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | latin1                     |
| character_set_connection | latin1                     |
| character_set_database   | utf8mb3                    |
| character_set_filesystem | binary                     |
| character_set_results    | latin1                     |
| character_set_server     | utf8mb4                    |
| character_set_system     | utf8mb3                    |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)

mysql> status
--------------
mysql  Ver 8.0.26-0ubuntu0.21.04.3 for Linux on aarch64 ((Ubuntu))

Connection id:          10
Current database:
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         8.0.26-0ubuntu0.21.04.3 (Ubuntu)
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    latin1
Conn.  characterset:    latin1
UNIX socket:            /var/run/mysqld/mysqld.sock
Binary data as:         Hexadecimal
Uptime:                 36 min 55 sec

Threads: 2  Questions: 325  Slow queries: 0  Opens: 181  Flush tables: 3  Open tables: 69  Queries per second avg: 0.146
--------------

mysql> 
```

MySQL 配置文件名称为 `my.cnf`，该文件在 MySQL 的安装目录下面。修改配置文件中的 `default-character-set` 和 `character-set-server` 参数值，将其改为想要的字符集名称，如 gbk、gb2312、latinl 等，修改完之后重新启动 MySQL 服务，即可生效。

```
## 找到 my.cnf 位置
root@modb:~# find /etc -iname my.cnf -print
/etc/alternatives/my.cnf
/etc/mysql/my.cnf

## 修改字符集
在[client ]下面加入
default-character-set=utf8
在[ mysqld ] 下面加
character_set_server=utf8

## 重启 mysql 生效
service mysql restart
```

此时，登录 MySQL 后使用 `SHOW VARIABLES LIKE 'character_set_%';` 或者 `status` 命令查看修改结果！

## 6、DISTINCT 可以应用于所有的列吗？

查询结果中，如果需要对列进行降序排序，可以使用 `DESC`，这个关键字只能对其前面的列 进行降序排列。

```
mysql> select * from lucifer;
+------+----------+
| id   | name     |
+------+----------+
|    1 | lucifer  |
|    2 | lucifer1 |
|    3 | lucifer2 |
+------+----------+
3 rows in set (0.00 sec)

mysql> select * from lucifer order by id desc;
+------+----------+
| id   | name     |
+------+----------+
|    3 | lucifer2 |
|    2 | lucifer1 |
|    1 | lucifer  |
+------+----------+
3 rows in set (0.00 sec)
```

例如，要对多列都进行降序排序，必须要在每一列的列名后面加 `DESC` 关键字。

```
mysql> select * from lucifer order by id desc,name desc;
+------+----------+
| id   | name     |
+------+----------+
|    3 | lucifer2 |
|    2 | lucifer1 |
|    1 | lucifer  |
+------+----------+
3 rows in set (0.00 sec)
```

而 `DISTINCT` 不同，DISTINCT 不能部分使用。换句话说，DISTINCT 关键字应用于所有列而不仅是它后面的第一个指定列。

例如，查询 2 个字段 sex，age，如果不同记录的这 2 个字段的组合值都不同，则所有记录都会被查询出来。

```
mysql> select * from lucifer;
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoli    | male   |   20 |
|    1 | xiaoliu   | female |   21 |
|    1 | xiaozhang | female |   21 |
|    1 | xiaowu    | female |   21 |
+------+-----------+--------+------+
4 rows in set (0.00 sec)

mysql> select distinct sex,age from lucifer;
+--------+------+
| sex    | age  |
+--------+------+
| male   |   20 |
| female |   21 |
+--------+------+
2 rows in set (0.00 sec)

mysql> 
```

## 7、ORDER BY 可以和 LIMIT 混合使用吗？

在使用 ORDER BY 子句时，应保证其位于 FROM 子句之后，如果使用 `LIMIT`，则必须位于 `ORDER BY` 之后，如果子句顺序不正确，MySQL 将产生错误消息。

**✅ 正确用法：**

```
mysql> select * from lucifer order by age desc limit 2,4;
+------+--------+--------+------+
| id   | name   | sex    | age  |
+------+--------+--------+------+
|    1 | xiaowu | female |   21 |
|    1 | xiaoli | male   |   20 |
+------+--------+--------+------+
2 rows in set (0.00 sec)
```

**❎ 错误用法：**

```
mysql> select * from lucifer limit 2,4 order by age desc;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'order by age desc' at line 1
mysql> 
```

## 8、什么时候使用引号？

在查询的时候，会看到在 WHERE 子句中使用条件，有的值加上了单引号，而有的值未加。

```
mysql> select * from lucifer where sex = 'female';
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoliu   | female |   21 |
|    1 | xiaozhang | female |   21 |
|    1 | xiaowu    | female |   21 |
+------+-----------+--------+------+
3 rows in set (0.00 sec)

mysql> 
```

**单引号用来限定字符串**，如果将值与字符串类型列进行比较，则需要限定引号；而用来与数值进行比较则不需要用引号。

```
mysql> select * from lucifer where age = 20;
+------+--------+------+------+
| id   | name   | sex  | age  |
+------+--------+------+------+
|    1 | xiaoli | male |   20 |
+------+--------+------+------+
1 row in set (0.00 sec)

mysql> 
```

## 9、在 WHERE子句中 AND 和 OR 必须使用圆括号吗？

任何时候使用具有 `AND` 和 `OR` 操作符的 `WHERE` 子句，都应该使用圆括号明确操作顺序。

```
mysql> select * from lucifer where (age = 20 or sex = 'female') and name != 'xiaowu';
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoli    | male   |   20 |
|    1 | xiaoliu   | female |   21 |
|    1 | xiaozhang | female |   21 |
+------+-----------+--------+------+
mysql> 3 rows in set (0.00 sec)
```

如果条件较多，即使能确定计算次序，默认的计算次序也可能会使 SQL 语句不易理解，因此使 用括号明确操作符的次序，是一个好的习惯。

## 10、更新或者删除表时必须指定 WHERE子 句吗？

个人建议所有的 UPDATE 和 DELETE 语句全都在 WHERE 子句中指定条件。

```
mysql> update lucifer set age = 22 where name = 'xiaoliu';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from lucifer where name = 'xiaoliu';
+------+---------+--------+------+
| id   | name    | sex    | age  |
+------+---------+--------+------+
|    1 | xiaoliu | female |   22 |
+------+---------+--------+------+
1 row in set (0.00 sec)

mysql> 
```

如果省略 WHERE 子句，则 UPDATE 或 DELETE 将被应用到表中所有的行。

```
mysql> update lucifer set age = 22;
Query OK, 3 rows affected (0.01 sec)
Rows matched: 4  Changed: 3  Warnings: 0

mysql> select * from lucifer;
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoli    | male   |   22 |
|    1 | xiaoliu   | female |   22 |
|    1 | xiaozhang | female |   22 |
|    1 | xiaowu    | female |   22 |
+------+-----------+--------+------+
4 rows in set (0.00 sec)

mysql> 
```

因此，除非确实打算更新或者删除所有记录，否则要注意使用不带 WHERE 子句的 UPDATE 或 DELETE  语句。

**📢 注意：** 建议在对表进行更新和删除操作之前，使用 SELECT 语句确认需要删除的记录，以免造成无法挽回的结果。

## 11、索引对数据库性能如此重要，应该如何使用它？

**索引的优点：**

- 通过创建唯一索引可以保证数据库表中每一行数据的唯一性。
- 可以给所有的 MySQL 列类型设置索引。
- 可以大大加快数据的查询速度，这是使用索引最主要的原因。
- 在实现数据的参考完整性方面可以加速表与表之间的连接。
- 在使用分组和排序子句进行数据查询时也可以显著减少查询中分组和排序的时间

**缺点：**

- 创建和维护索引组要耗费时间，并且随着数据量的增加所耗费的时间也会增加。
- 索引需要占磁盘空间，除了数据表占数据空间以外，每一个索引还要占一定的物理空间。如果有大量的索引，索引文件可能比数据文件更快达到最大文件尺寸。
- 当对表中的数据进行增加、删除和修改的时候，索引也要动态维护，这样就降低了数据的维护速度。

使用索引时，需要综合考虑索引的优点和缺点。

为数据库选择正确的索引是一项复杂的任务。如果索引列较少，则需要的磁盘空间和维护开销 都较少。如果在一个大表上创建了多种组合索引，索引文件也会膨胀很快。

而另一方面，索引较多 可覆盖更多的查询。可能需要试验若干不同的设计，才能找到最有效的索引。可以添加、修改和删 除索引而不影响数据库架构或应用程序设计。

因此，应尝试多个不同的索引从而建立最优的索引。

## 12、尽量使用短索引（前缀索引）

对字符串类型的字段进行索引，如果可能应该指定一个前缀长度。

例如，如果有一个  CHAR(255) 的列，如果在前 10 个或 30 个字符内，多数值是惟一的，则不需要对整个列进行索引。

```
mysql> select * from lucifer;
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoli    | male   |   22 |
|    1 | xiaoliu   | female |   22 |
|    1 | xiaozhang | female |   22 |
|    1 | xiaowu    | female |   22 |
+------+-----------+--------+------+
4 rows in set (0.00 sec)

mysql> create index idx_lucifer_name on lucifer (name(4));
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show index from lucifer;
+---------+------------+------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table   | Non_unique | Key_name         | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+---------+------------+------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| lucifer |          1 | idx_lucifer_name |            1 | name        | A         |           1 |        4 |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
+---------+------------+------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
1 row in set (0.01 sec)

mysql> 
```

**短索引不仅可以提高查询速度而且可以节省磁盘空间、减少 I/O 操作。**

## 13、MySQL 存储过程和函数有什么区别？

在本质上它们都是存储程序。

**函数：**

- 只能通过 return 语句返回单个值或者表对象；
- 限制比较多，不能用临时表，只能用表变量，还有一些函数都不可用等等；
- 可以嵌入在 SQL 语句中使用，可以在 SELECT 语句中作为查询语句的一个部分调用；

**存储过程：**

- 不允许执行 return，但是可以通过 out 参数返回多个值；
- 限制相对就比较少；
- 一般是作为一个独立的部分来执行；

## 14、存储过程中的内容可以改变吗？

**不可以！**

目前，MySQL 还不提供对已存在的存储过程代码的修改，如果必须要修改存储过程，必须使用 DROP 语句删除之后，再重新编写代码，或者创建一个新的存储过程。

**不得不说，这方面还是 Oracle 做的比较好。**

## 15、存储过程中可以调用其他存储过程吗？

**可以！**

存储过程包含用户定义的 SQL 语句集合，可以使用 CALL 语句调用存储过程，当然在存储过程中也可以使用 CALL 语句调用其他存储过程，但是不能使用 DROP 语句删除其他存储过程。

## 16、存储过程的参数不要与数据表中的字段名相同。

在定义存储过程参数列表时，应注意把参数名与数据库表中的字段名区别开来，否则将出 现无法预期的结果。

## 17、存储过程的参数可以使用中文吗？

一般情况下，可能会出现存储过程中传入中文参数的情况，例如某个存储过程根据用户的 名字查找该用户的信息，传入的参数值可能是中文。这时需要在定义存储过程的时候，在后面加 上 character set gbk,不然调用存储过程使用中文参数会出错，比如定义 userInfo 存储过程，代码 如下：

CREATE PROCEDURE useInfo(IN u_name VARCHAR(50) character set gbk, OUT u_age INT)

## 18、MySQL 中视图和表的区别以及联系是什么？

**两者的区别：**

- 视图是已经编译好的 SQL 语句，是基于 SQL 语句的结果集的可视化的表，而表不是；
- 视图没有实际的物理记录，而基本表有；
- 表是内容，视图是窗口；
- 表占用物理空间而视图不占用物理空间，视图只是逻辑概念的存在，表可以及时对它进行修改，但视图只能用创建的语句来修改；
- 视图是查看数据表的一种方法，可以查询数据表中某些字段构成的数据，只是一些SQL 语句的集合。从安全的角度来说，视图可以防止用户接触数据表，因而用户不知道表结构；
- 表属于全局模式中的表，是实表；视图属于局部模式的表，是虚表；
- 视图的建立和删除只影响视图本身，不影响对应的基本表；

**两者的联系：**

视图（view）是在基本表之上建立的表，它的结构（即所定义的列）和内容（即所有记录） 都来自基本表，它依据基本表存在而存在。

一个视图可以对应一个基本表，也可以对应多个基本表。

视图是基本表的抽象和在逻辑意义上建立的新关系。

## 19、使用触发器时须特别注意！

在使用触发器的时候需要注意，对于相同的表，相同的事件只能创建一个触发器。

```
mysql> create trigger lucifer_tri before insert on lucifer for each row set NEW.id=NEW.id+1;
Query OK, 0 rows affected (0.01 sec)

mysql> 
mysql> 
mysql> select * from lucifer;
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoli    | male   |   22 |
|    1 | xiaoliu   | female |   22 |
|    1 | xiaozhang | female |   22 |
|    1 | xiaowu    | female |   22 |
|    1 | lucifer   | male   |   20 |
|    1 | lucifer   | male   |   20 |
+------+-----------+--------+------+
6 rows in set (0.00 sec)

mysql> insert into lucifer values(1,'lucifer','male',20);
Query OK, 1 row affected (0.00 sec)

mysql> select * from lucifer;
+------+-----------+--------+------+
| id   | name      | sex    | age  |
+------+-----------+--------+------+
|    1 | xiaoli    | male   |   22 |
|    1 | xiaoliu   | female |   22 |
|    1 | xiaozhang | female |   22 |
|    1 | xiaowu    | female |   22 |
|    1 | lucifer   | male   |   20 |
|    1 | lucifer   | male   |   20 |
|    2 | lucifer   | male   |   20 |
+------+-----------+--------+------+
7 rows in set (0.00 sec)
```

比如对表 lucifer 创建了一个 `BEFORE INSERT` 触发器，那么如果对表 lucifer 再次创建一个 `BEFORE INSERT` 触发器，MySQL 将会报错，此时，只可以在表 lucifer 上创建 `AFTER INSERT` 或者 `BEFORE UPDATE` 类型的触发器。

```
mysql> create trigger lucifer_tri before insert on lucifer for each row set NEW.id=NEW.id+1;
ERROR 1359 (HY000): Trigger already exists
mysql> 
```

灵活的运用触发器将为操作省去很多麻烦。

## 20、及时删除不再需要的触发器

触发器定义之后，每次执行触发事件，都会激活触发器并执行触发器中的语句。

如果需求发生变化，而触发器没有进行相应的改变或者删除，则触发器仍然会执行旧的语句，从而会影响新的数据的完整性。

```
mysql> drop trigger lucifer_tri;
Query OK, 0 rows affected (0.03 sec)

mysql> 
```

因此，要将不再使用的触发器及时删除。

## 21、应该使用哪种方法创建用户？（3种方式）

创建用户有 3 种方法：

- 使用 CREATE USER 语句创建用户
- 在 mysql.user 表中添加用户
- 使用 GRANT 语句创建用户（仅限 MySQL 8 版本以下使用）

一般情况， 最好使用 GRANT 或者 CREATE USER 语句，而不要直接将用户信息插入 user 表，因为 user 表中存储了全局级别的权限以及其他的账户信息，如果意外破坏了 user 表中的记录，则可能会对  MySQL 服务器造成很大影响。

```
-- 使用 CREATE USER 语句创建用户
mysql> create user 'lucifer'@'localhost' identified by 'lucifer';
Query OK, 0 rows affected (0.01 sec)

mysql> 

-- 在 mysql.user 表中添加用户
mysql> select MD5('lucifer');
+----------------------------------+
| MD5('lucifer')                   |
+----------------------------------+
| cae33a0264ead2ddfbc3ea113da66790 |
+----------------------------------+
1 row in set (0.00 sec)

mysql> 
mysql> INSERT INTO mysql.user(Host, User, authentication_string, ssl_cipher, ssuex509_i09_sr, x5ubject) VALUES ('lohoscalt',uci 'lfer MD5('1',lucifer'), '', '',; '')
Query OK, 1 row affected (0.01 sec)

mysql> 

-- 使用 GRANT 语句创建用户
mysql> GRANT SELECT ON*.* TO 'lucifer2'@localhost IDENTIFIED BY 'lucifer';
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'IDENTIFIED BY 'lucifer'' at line 1
mysql>
```

**📢 注意：** 由于测试使用的是 MySQL 8 版本，已经不支持 GRANT 直接创建用户，5.7 版本依然是支持的。

## 22、mysqldump 备份的文件只能在 MySQL 中使用吗？

逻辑备份工具，适用于所有的存储引擎，支持温备、完全备份、部分备份、对于 InnoDB 存储引擎支持热备。

`mysqldump` 备份的文本文件实际是数据库的一个副本，使用该文件不仅可以在 MySQL 中恢复数据库，而且通过对该文件的简单修改，可以使用该文件在 SQL Server 或者 Sybase 等其他数据库中恢复数据库。

```
root@modb:~# mysqldump -uroot -p hr > /root/hr.db
Enter password: 
root@modb:~# 
root@modb:~# ll hr.db 
-rw-r--r-- 1 root root 25327 Nov 26 08:52 hr.db
```

这在某种程度上实现了数据库之间的迁移。

## 23、如何选择备份工具？

根据备份的方法（是否需要数据库离线）可以将备份分为：

- 热备（Hot Backup）
- 冷备（Cold Backup）
- 温备（Warm Backup）

MySQL 中进行不同方式的备份还要考虑存储引擎是否支持，如 MyISAM 不支持热备，支持温备和冷备。而 InnoDB 支持热备、温备和冷备。

一般情况下，我们需要备份的数据分为以下几种：

- 表数据
- 二进制日志、InnoDB 事务日志
- 代码（存储过程、存储函数、触发器、事件调度器）
- 服务器配置文件

下面是几种常用的备份工具：

- mysqldump：逻辑备份工具，适用于所有的存储引擎，支持温备、完全备份、部分备份、对于 InnoDB 存储引擎支持热备。
- cp、tar 等归档复制工具：物理备份工具，适用于所有的存储引擎、冷备、完全备份、部分备份。
- lvm2 snapshot：借助文件系统管理工具进行备份。
- mysqlhotcopy：名不副实的一个工具，仅支持 MyISAM 存储引擎。
- xtrabackup：一款由 percona 提供的非常强大的 InnoDB/XtraDB 热备工具，支持完全备份、增量备份。

直接复制数据文件是最为直接、快速的备份方法，但缺点是基本上不能实现增量备份。备份时必须确保没有使用这些表。如果在复制一个表的同时服务器正在修改它，则复制无效。备份 文件时，最好关闭服务器，然后重新启动服务器。

## 24、平时应该打开哪些日志？

日志既会影响 MySQL 的性能，又会占用大量磁盘空间。因此，如果不必要，应尽可能少地 开启日志。

根据不同的使用环境，可以考虑开启不同的日志。

例如，在开发环境中优化查询效率低的语句，可以开启慢查询日志；

**开启慢查询日志：** 可以让MySQL记录下查询超过指定时间的语句，通过定位分析性能的瓶颈，才能更好的优化数据库系统的性能。

```
-- 检查是否开启慢查询
mysql> show variables like 'slow_query%';
+---------------------+------------------------------+
| Variable_name       | Value                        |
+---------------------+------------------------------+
| slow_query_log      | OFF                          |
| slow_query_log_file | /var/lib/mysql/modb-slow.log |
+---------------------+------------------------------+
2 rows in set (0.00 sec)

mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.01 sec)

-- 开启慢查询日志
mysql> set global slow_query_log='ON'; 
Query OK, 0 rows affected (0.00 sec)

-- 设置查询超过10秒就记录
mysql> set global long_query_time=10;
Query OK, 0 rows affected (0.00 sec)

-- 再次检查是否开启
mysql> show variables like 'slow_query%';
mysql> +---------------------+------------------------------+
| Variable_name       | Value                        |
+---------------------+------------------------------+
| slow_query_log      | ON                           |
| slow_query_log_file | /var/lib/mysql/modb-slow.log |
+---------------------+------------------------------+
2 rows in set (0.00 sec)
```

如果需要记录用户的所有查询操作，可以开启通用查询日志；

```
mysql> show variables like 'general_log%';
+------------------+-------------------------+
| Variable_name    | Value                   |
+------------------+-------------------------+
| general_log      | OFF                     |
| general_log_file | /var/lib/mysql/modb.log |
+------------------+-------------------------+
2 rows in set (0.00 sec)

-- 开启通用查询日志
mysql> SET GLOBAL general_log=1; 
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'general_log%';
+------------------+-------------------------+
| Variable_name    | Value                   |
+------------------+-------------------------+
| general_log      | ON                      |
| general_log_file | /var/lib/mysql/modb.log |
+------------------+-------------------------+
2 rows in set (0.00 sec)
```

如果需要记录数据的变更，可以开启二进制日志；错误日志是默认开启的。

```
mysql> show variables like 'log_bin%';
+---------------------------------+-----------------------------+
| Variable_name                   | Value                       |
+---------------------------------+-----------------------------+
| log_bin                         | ON                          |
| log_bin_basename                | /var/lib/mysql/binlog       |
| log_bin_index                   | /var/lib/mysql/binlog.index |
| log_bin_trust_function_creators | OFF                         |
| log_bin_use_v1_row_events       | OFF                         |
+---------------------------------+-----------------------------+
5 rows in set (0.00 sec)

mysql> 
```

## 25、如何使用二进制日志？

二进制日志主要用来记录数据变更。

如果需要记录数据库的变化，可以开启二进制日志。基于二进制日志的特性，不仅可以用来进行数据恢复，还可用于数据复制。

```
root@modb:/var/lib/mysql# ls binlog*
binlog.000001  binlog.000002  binlog.index
root@modb:/var/lib/mysql# mysqlbinlog binlog.000001 | mysql -u root -p                                            
Enter password: 
root@modb:/var/lib/mysql# 
```

在数据库定期备份的 情况下，如果出现数据丢失，可以先用备份恢复大部分数据，然后使用二进制日志恢复最近备份后变更的数据。在双机热备情况下，可以使用 MySQL 的二进制日志记录数据的变更，然后将变更部分复制到备份服务器上。

## 26、如何使用慢查询日志？

慢查询日志主要用来记录查询时间较长的日志。

在开发环境下，可以开启慢查询日志来记录查询时间较长的查询语句，然后对这些语句进行优化。

```
root@modb:/var/lib/mysql# cat /var/lib/mysql/modb-slow.log
/usr/sbin/mysqld, Version: 8.0.26-0ubuntu0.21.04.3 ((Ubuntu)). started with:
Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
Time                 Id Command    Argument
root@modb:/var/lib/mysql# 
```

通过配 `long_query_time` 的值，可以灵活地掌握不同程度的慢查询语句。

## 27、是不是索引建立得越多越好？

合理的索引可以提高查询的速度，但不是索引越多越好。

在执行插入语句的时候，MySQL 要为新插入的记录建立索引。所以过多的索引会导致插入操作变慢。原则上是只有查询用的字段才建立索引。

使用索引时，需要综合考虑索引的优点和缺点。

## 28、如何使用查询缓冲区？

查询缓冲区可以提高查询的速度，但是这种方式只适合查询语句比较多、更新语句比较少 的情况。

默认情况下查询缓冲区的大小为 0，也就是不可用。可以修改 `queiy_cache_size` 以调整查询缓冲区大小；修改 `query_cache_type` 以调整查询缓冲区的类型。

在 `my.cnf` 中修改 `query_cache_size` 和 `query_cache_type` 的值如下所示：

```
[mysqld]
query_cache_size= 512M 
query_cache_type= 1
query_cache_type=1
```

表示开启查询缓冲区。

只有在查询语句中包含 `SQL_NO_CACHE` 关键字时，才不会使用查询缓冲区。可以使用 `FLUSH QUERY CACHE` 语句来刷新缓冲区，清理查询缓冲区中的碎片。