使用命令发现磁盘使用率为 100% 了，还剩几十兆。

## 神操作
备份数据库，删除实例、删除数据库表、重启 mysql 服务.结果磁盘空间均为释放

## 怎么办
网上查了很多资源，说要进行磁盘碎片化整理。原因是 datafree 占据的空间太多啦。具体可以通过这个 sql 查看。
```sql
SELECT CONCAT(TRUNCATE(SUM(data_length)/1024/1024,2),'MB') AS data_size,
CONCAT(TRUNCATE(SUM(max_data_length)/1024/1024,2),'MB') AS max_data_size,
CONCAT(TRUNCATE(SUM(data_free)/1024/1024,2),'MB') AS data_free,
CONCAT(TRUNCATE(SUM(index_length)/1024/1024,2),'MB') AS index_size
FROM information_schema.tables WHERE TABLE_NAME = 'datainfo';
```
这个是后来的图了，之前的图没有留，当时显示一张表里的 data_free 都达到了 20 个 G。

![1622358553029](C:\Users\ADMINI~1\AppData\Local\Temp\1622358553029.png)

网上推荐的做法如下所示，对表格进行碎片化整理。

```sql
ALTER TABLE datainfo ENGINE=InnoDB;ANALYZE TABLE datainfo;optimize table datainfo;
```

## 僵局：

查看数据库版本为 5.562 不支持 inodb，要么选择升级数据库。正在这时，有个不好的消息发生了，那张表格给删掉了，但是磁盘空间还是没有释放啊。所以对表进行碎片化整理的路也走不通了，因为表没了。。。

后来的神操作
1、使用命令查看 mysql 安装的位置和配置文件所在的地方

```
mysql 1118 945 0 14:28 ? 00:00:00 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --log-error=/var/log/mysqld.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/var/lib/mysql/mysql.sock
```

2、关闭 mysql
`service mysql stop　　`

3、删除 `datadir `目录下的 `ibdata1`、`ib_logfile0 ib_logfile1 `这些文件


4、 移动 `mysql `的启动参数
`mv /etc/my.cnf ./abc`

5、重新启动 mysql 发现磁盘空间释放了
`service mysql start`

磁盘空间终于释放了

### 下一步数据库还原

1、采用 navicate 备份工具，进行数据库备份


备份成功后生成了，生成 psc 文件。
200409141055.psc

2、新建一个数据库实例，设置数据库名和字符集


3、然后对备份数据库进行还原，点击还原


4、开始进行还原
第一次还原后发现还原后数据库表建成功了，但是表里面没有数据。
后来网上查找资料发现是，遇到错误就停止了。所以更改了还原的配置，再次进行还原。
之前是这样设置的


还原时当成一个事务进行了，遇到错误就停止了。
更改配置


重新进行还原，数据库里的数据有了，并且验证没有问题。

## 问题解决

### mysql 碎片化产生的原因

（1）表的存储会出现碎片化，每当删除了一行内容，该段空间就会变为被留空，而在一段时间内的大量删除操作，会使这种留空的空间变得比存储列表内容所使用的空间更大；

（2）当执行插入操作时，MySQL 会尝试使用空白空间，但如果某个空白空间一直没有被大小合适的数据占用，仍然无法将其彻底占用，就形成了碎片；

（3）当 MySQL 对数据进行扫描时，它扫描的对象实际是列表的容量需求上限，也就是数据被写入的区域中处于峰值位置的部分；

### 清除碎片的优点:

降低访问表时的 IO，提高 mysql 性能,释放表空间降低磁盘空间使用率

## 注意

1.MySQL 官方建议不要经常 (每小时或每天) 进行碎片整理，一般根据实际情况，只需要每周或者每月整理一次即可 (我们现在是每月凌晨 4 点清理 mysql 所有实例下的表碎片)。
2.在 OPTIMIZE TABLE 运行过程中，MySQL 会锁定表。因此，这个操作一定要在网站访问量较少的时间段进行。
3.清理 student 的 105 万条数据, OPTIMIZE TABLE 库.student;本地测试需要 37 秒。

## 自测

大家可以用这条语句看看自己的系统的 datafree 大不大：`show table status from 表名`。