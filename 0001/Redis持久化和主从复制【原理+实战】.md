Redis持久化和主从复制【原理+实战】

## 一、持久化

所谓的持久化就是把内存中的数据写到磁盘中去，防止服务宕机后内存数据丢失。Redis4.0之前提供了两种持久化方式:RDB（默认） 和AOF，Redis4.x之后新增了一种混合持久化（本文所用的Redis版本是**redis‐5.0.2**）

### 1、RDB

RDB是Redis Database缩写，在默认情况下，Redis将内存数据库快照保存在名字为dump.rdb的二进制文件中。可以对Redis进行设置，让它在**“** N秒内至少有M个键值改动**”**这一条件被满足时，自动保存一次数据。比如下图，900秒内有1个键值或者300秒内有10个键值或者60秒内有10000个键值改动，自动保存一次数据；关闭RDB只需要将所有的save保存策略注释掉即可。

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202212508702-239023260.png)

还可以手动执行命令生成RDB快照，进入Redis客户端执行命令save或bgsave可以生成dump.rdb文件，每次命令执行都会将所有Redis内存快照到一个新的rdb文件里，并覆盖原有rdb快照文件。save是同步命令，bgsave是异步命令，bgsave会从redis主进程fork（fork()是linux函数）出一个子进程专门用来生成rdb快照文件。Redis配置自动生成rdb文件后台使用的是bgsave方式。

| 命令                  | save             | bgsave                                        |
| --------------------- | ---------------- | --------------------------------------------- |
| IO类型                | 同步             | 异步                                          |
| 是否阻塞redis其它命令 | 是               | 否(在生成子进程执行调用fork函数时会短暂阻塞） |
| 复杂度                | O(n)             | O(n)                                          |
| 优点                  | 不会消耗额外内存 | 不阻塞客户端命令                              |
| 缺点                  | 阻塞客户端命令   | 需要fork子进程，消耗内存                      |

### 2、AOF

AOF是append-only file缩写，RDB快照并不是非常耐久（durable）：如果Redis因为某些原因而造成故障停机，那么服务器将丢失最近写入、且仍未保存到快照中的那些数据。从Redis1.1版本开始，Redis增加了一种完全耐久的持久化方式：AOF持久化。可以通过修改如下配置文件来打开AOF功能：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202215031852-1137682862.png)

修改了配置文件，先执行bin/redis-cli shutdown停止Redis，然后执行bin/redis-server redis.conf启动Redis，此时appendonly生效；从现在开始， 每当Redis执行一个改变键值的命令时（比如 SET），这个命令就会被追加到AOF文件的末尾。这样的话，当 Redis重新启动时，程序就可以通过重新执行AOF文件中的命令来达到重建数据的目的。你可以配置 Redis 多久才将数据 fsync到磁盘一次。

- `appendfsync always`：每次有新命令追加到AOF文件时就执行一次fsync，非常慢，也非常安全。
- `appendfsync everysec`：每秒fsync一次，足够快（和使用 RDB 持久化差不多），并且在故障时只会丢失 1 秒钟的数据。
-  `appendfsync no`：从不 fsync，将数据交给操作系统来处理。更快，也更不安全的选择。

**推荐（并且也是默认）的措施为每秒fsync一次，这种fsync策略可以兼顾速度和安全性。配置文件如下：**

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202221926361-423621772.png)

#### 执行如下命令：

（1）启动客户端，连接Redis bin/redis-cli 并执行set toby xu![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202220121994-542179969.png)

(2) 到dir（redis.conf这个配置文件里面的数据持久化的目录属性）所在的目录下查看，如下图：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202220025983-1201495957.png)

（3）`vim appendonly.aof`，文件的内容在后面的RESP（Redis序列化协议）中详解讲解，Redis序列化协议官网地址：`https://redis.io/topics/protocol`

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202220453930-1061763838.png)

####  AOF重写：

（1）AOF文件里可能有太多没用指令，所以AOF会定期根据内存的最新数据生成新的aof文件，当然可以手工执行**bgrewriteaof**命令也能重写AOF，比如执行如下命令：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202223712862-1126592925.png)

（2）重写后AOF文件里变成：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202224028516-85224917.png)

如下两个配置可以控制AOF自动重写频率：

　　**① auto-aof-rewrite-min-size 64mb ：**aof文件至少要达到64M才会自动重写。

　　**② auto-aof-rewrite-percentage 100 ：**aof文件自上一次重写后文件大小增长了100%则再次触发重写。

当然AOF还可以手动重写，进入redis客户端执行如上图命令bgrewriteaof重写AOF注意，AOF重写Redis会fork出一个子进程去做，不会对Redis正常命令处理有太多影响。

### 3、RDB和AOF对比

Redis启动时如果既有RDB文件又有AOF文件则优先选择AOF文件恢复数据，因为AOF一般来说数据更全一点。

| 持久化方式 | RDB        | AOF          |
| ---------- | ---------- | ------------ |
| 启动优先级 | 低         | 高           |
| 文件大小   | 小         | 大           |
| 恢复速度   | 快         | 慢           |
| 数据安全性 | 容易丢数据 | 根据策略决定 |

### 4、Redis4.0混合持久化

重启Redis时，我们很少使用 RDB来恢复内存状态，因为会丢失大量数据。我们通常使用AOF日志重放，但是重放AOF日志性能相对RDB来说要慢很多，这样在 Redis 实例很大的情况下，启动需要花费很长的时间。 Redis4.0为了解决这个问题，带来了一个新的持久化选项——混合持久化。配置如下：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202225851783-1435672658.png)

如果开启了混合持久化，AOF在重写时，不再是单纯将内存数据转换为RESP命令写入AOF文件，而是将重写这一刻之前的内存做RDB快照处理，并且将RDB快照内容和增量的AOF修改内存数据的命令存在一起，都写入新的AOF文件，新的文件一开始不叫appendonly.aof，等到重写完新的AOF文件才会进行改名，原子的覆盖原有的AOF文件，完成新旧两个AOF文件的替换。于是在Redis重启的时候，可以先加载RDB的内容，然后再重放增量AOF文件就可以完全替代之前的AOF全量文件重放，因此重启效率大幅得到提升。

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202230659250-1867865220.png)

##  二、Redis主从

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191202233143319-1760744854.png)

### 1、主从复制概念

主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master)，后者称为从节点(slave),数据的复制是单向的，只能由主节点到从节点。

### 2、主从复制的原理

#### （1）全量复制

将主节点中的所有数据都发送给从节点，是一个非常重型的操作，当数据量较大时，会对主从节点和网络造成很大的开销。全量复制流程图如下：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191203191832918-402540056.png)

　　① slave会发出一个同步命令，刚开始是Psync命令，表示要求master主机同步数据

　　② master收到psync命令后，会通过执行bgsave生成最新的RDB快照文件，持久化期间，master会继续接收客户端的请求，它会把写请求缓存在内存中

　　③ 发送RDB文件给slave

　　④ master再将之前缓存在内存中的命令发送给slave

　　⑤ 刷新旧的数据。slave在载入主节点的数据之前要先将老数据清除

　　⑥ 加载RDB文件将数据库状态更新至主节点执行bgsave时的数据库状态和缓冲区数据的加载

　　⑦ master同步长连接持续把写命令发送给slave，以保证数据的一致

#### （2）部分复制

部分复制是Redis 2.8以后出现的，用于处理在主从复制中因网络闪断等原因造成的数据丢失场景，当slave再次连上master后，如果条件允许，master会补发丢失数据给slave。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销。部分复制流程图如下：

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191203194155935-1124107934.png)

　　① 如果网络抖动（连接断开 connection lost）

　　② master还是会写repl_back_buffer（复制缓冲区）

　　③ slave会继续尝试连接主机

　　④ slave会把自己当前run_id和偏移量传输给master，并且执行pysnc命令同步

　　⑤ slave发送过来的offset在repl_back_buffer中，则master会将缓存中从offset以后的数据一次性同步给slave，否则全量复制

　　⑥ master同步长连接持续把写命令发送给slave，以保证数据的一致

### 3、主从搭建

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191203194740851-19885114.png)

 其中slave的主要配置如下：

```
port 6380
pidfile /var/run/redis_6380.pid
dir /usr/local/redis-5.0.2/6380
replicaof 192.168.160.146 6379
replica-serve-stale-data yes
replica-read-only yes
```

（1）在`6379 set toby xu `

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191203201035340-329605882.png)

 （2）在`6380 keys *`

![img](https://img2018.cnblogs.com/i-beta/1761778/201912/1761778-20191203201104619-1238498589.png)

 至此Redis主从搭建完成！！！