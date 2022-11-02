面试官：说说`MySQL`存储引擎原理

`MySQL`的数据是如何组织的呢？当然是page，也就是说`MySQL`以页为单位进行内外存交换。

## 一、 MySQL记录存储（页为单位）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531195507388.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 



### 页头

记录页面的控制信息，共占56字节，包括页的左右兄弟页面指针、页面空间使用情况等。

### 虚记录

最大虚记录：比页内最大主键还大
最小虚记录：比页内最小主键还小
(作用：比如说我们要查看一个记录是否在这个页面里，就要看这个记录是否在最大最小虚记录范围内)

### 记录堆

行记录存储区，分为有效记录和已删除记录两种

### 自由空间链表

已删除记录组成的链表
(重复利用空间)

### 未分配空间

页面未使用的存储空间；

### Slot区

页尾
页面最后部分，占8个字节，主要存储页面的校验信息； 

### 页内记录维护

**顺序保证** 

物理有序(利于查询，不利于插入删除) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531195527636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

**逻辑有序(插入删除性能高，查询效率低)** 默认 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531195544502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

所以`MySQL`是像下图所示这样子有序的组织数据的。 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531195631224.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

**2、插入策略** 

**自由空间链表**（优先利用自由空间链表）

**未使用空间**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531195727657.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70)

**3、页内查询** 

**遍历**

**二分查找(数据不一样大，不能用二分)**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531200200482.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

利用槽位做二分，实现近似的二分查找，近似于跳表 。

## 二、 MySQL InnoDB存储引擎内存管理

### 预分配内存空间

内存池

### 数据以页为单位加载 （减少io访问次数）

内存页面管理

- 页面映射（记录哪块磁盘上的数据加载到哪块内存上了）
- 页面数据管理

### 数据内外存交换

数据淘汰

- 内存页耗尽
- 需要加载新数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531200353108.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

### 页面管理

- 空闲页
- 数据页
- 脏页（需刷回磁盘）

### 页面淘汰

LRU(淘汰冷数据)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021053120082160.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 


`某时刻状态->访问P2->访问新页P7`

#### 全表扫描对内存的影响？

可能会把内存中的热数据淘汰掉（比如说对一个几乎没有访问量的表进行全表扫描）

所以`MySQL`不是单纯的利用`LRU算法`

解决问题：如何避免热数据被淘汰？

解决方案：
访问时间 + 频率(redis)

两个`LRU`表

`MySQL`的解决方案

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531225157238.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

MySQL内存管理—LRU

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531201435871.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

### 页面装载

磁盘数据到内存

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531201748592.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531214131873.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

没有空闲页怎么办？
`Free list中取 > LRU中淘汰 > LRU Flush`

### 页面淘汰

`LRU`尾部淘汰
`Flush LRU`淘汰

`LRU`链表中将第一个脏页刷盘并“释放”，放到`LRU`尾部？直接放`FreeList`？

### 位置移动

old 到 new
new 到 old

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531214410169.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

思考：移动时机是什么？
`innodb_old_blocks_time`
old区存活时间，大于此值，有机会进入new区

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531214516414.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

`LRU_new`的操作
链表操作效率很高，有访问移动到表头？
`Lock！！！`
`MySQL`设计思路： 减少移动次数

两个重要参考：
1、`freed_page_clock：Buffer Pool`淘汰页数
2、`LRU_new`长度1/4

当前`freed_page_clock` - 上次移动到Header时`freed_page_clock>LRU_new长度1/4`

## 三、MySQL事务实现原理

MySQL事务基本概念

### 1、事务特性

A（`Atomicity`原子性）：全部成功或全部失败
I（`Isolation`隔离性）：并行事务之间互不干扰
D（`Durability`持久性）：事务提交后，永久生效
C（`Consistency`一致性）：通过AID保证

### 2、并发问题

脏读(`Drity Read`)：读取到未提交的数据
不可重复读(`Non-repeatable read`)：两次读取结果不同
幻读(`Phantom Read`)：select 操作得到的结果所表征的数据状态无法支撑后续的业务操作

### 3、隔离级别

`Read Uncommitted`（未提交读）：最低隔离级别，会读取到其他事务未提交的数据。脏读；
`Read Committed`（提交读）：事务过程中可以读取到其他事务已提交的数据。不可重复读；
`Repeatable Read`（可重复读）：每次读取相同结果集，不管其他事务是否提交，幻读；（两次当前读不会产生幻读）
`Serializable`（串行化）：事务排队，隔离级别最高，性能最差；

### MySQL事务实现原理（事务管理机制）

#### 1、MVCC 多版本并发控制

解决读-写冲突
如何工作：隐藏列

–当前读（读在存储引擎中存储的那个数据）

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021053121542425.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

RR级别下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531215601263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

#### 2、undo log

回滚日志
保证事务原子性
实现数据多版本
`delete undo log`：用于回滚，提交即清理；
`update undo log`：用于回滚，同时实现快照读，不能随便删除

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531215651361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

思考：undolog如何清理？
依据系统活跃的最小活跃事务ID Read view
为什么InnoDB count（*）这么慢？
因为

#### 3、redo log

实现事务持久性

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531215927186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 


写入流程
l 记录页的修改，状态为prepare
l 事务提交，讲事务记录为commit状态

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220003217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220022409.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

#### 意义

体积小，记录页的修改，比写入页代价低
末尾追加，随机写变顺序写，发生改变的页不固定

## 四、MySQL锁实现原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220209156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

所有当前读加排他锁，都有哪些是当前读？
`SELECT FOR UPDATE`
`UPDATE`
`DELETE`

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220317605.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220340936.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

唯一索引/非唯一索引 `* RC/RR`
4种情况逐一分析

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220433147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 


会出现幻读问题，不可重复读了

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220454755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 



![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220524616.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220547363.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220640221.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220700273.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210531220724587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjI2OTI1Nw==,size_16,color_FFFFFF,t_70) 



死锁在库表中有记录，通过kill 那个锁删除。