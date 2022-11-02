## 面试数十家Linux运维工程师，总结了这些面试题（含答案）

下面是一名资深Linux运维求职数十家公司总结的Linux运维面试精华，助力大家跳槽找个高薪好工作。

#### 1、什么是运维？什么是游戏运维？

1）运维是指大型组织已经建立好的网络软硬件的维护，就是要保证业务的上线与运作的正常，
在他运转的过程中，对他进行维护，他集合了网络、系统、数据库、开发、安全、监控于一身的技术
运维又包括很多种，有DBA运维、网站运维、虚拟化运维、监控运维、游戏运维等等

2）游戏运维又有分工，分为开发运维、应用运维（业务运维）和系统运维
开发运维：是给应用运维开发运维工具和运维平台的
应用运维：是给业务上线、维护和做故障排除的，用开发运维开发出来的工具给业务上线、维护、做故障排查
系统运维：是给应用运维提供业务上的基础设施，比如：系统、网络、监控、硬件等等

总结：开发运维和系统运维给应用运维提供了“工具”和“基础设施”上的支撑
开发运维、应用运维和系统运维他们的工作是环环相扣的

#### 2、在工作中，运维人员经常需要跟运营人员打交道，请问运营人员是做什么工作的？

游戏运营要做的一个事情除了协调工作以外
还需要与各平台沟通，做好开服的时间、开服数、用户导量、活动等计划

#### 3、现在给你三百台服务器，你怎么对他们进行管理？

管理3百台服务器的方式：
1）设定跳板机，使用统一账号登录，便于安全与登录的考量。
2）使用[salt](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247487168&idx=1&sn=28114b11c2279546271e3f43d69d7a16&chksm=9ac54d4aadb2c45ca738a5952c4186381f733ce8bd36223c0ee0c15c1a83ced40200c342fc5d&scene=21#wechat_redirect)、[ansible](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247507445&idx=1&sn=d7cac888030bb31141f28f159df39f6e&chksm=9ac69e7fadb117697e87d0b4023ff58f5241c7307729f036b05d64832336571116ae4c9bbb0f&scene=21#wechat_redirect)、puppet进行系统的统一调度与配置的统一管理。
3）建立简单的服务器的系统、配置、应用的cmdb信息管理。便于查阅每台服务器上的各种信息记录。

#### 4、简述raid0 raid1 raid5 三种工作模式的工作原理及特点

RAID，可以把硬盘整合成一个大磁盘，还可以在大磁盘上再分区，放数据
还有一个大功能，多块盘放在一起可以有冗余（备份）
RAID整合方式有很多，常用的：0 1 5 10

RAID 0，可以是一块盘和N个盘组合
其优点读写快，是RAID中最好的
缺点：没有冗余，一块坏了数据就全没有了

RAID 1，只能2块盘，盘的大小可以不一样，以小的为准
10G+10G只有10G，另一个做备份。它有100%的冗余，缺点：浪费资源，成本高

RAID 5 ，3块盘，容量计算10*（n-1）,损失一块盘
特点，读写性能一般，读还好一点，写不好

冗余从好到坏：RAID1 RAID10 RAID 5 RAID0
性能从好到坏：RAID0 RAID10 RAID5 RAID1
成本从低到高：RAID0 RAID5 RAID1 RAID10

单台服务器：很重要盘不多，系统盘，RAID1
数据库服务器：主库：RAID10 从库 RAID5\RAID0（为了维护成本，RAID10）
WEB服务器，如果没有太多的数据的话，RAID5,RAID0（单盘）
有多台，监控、应用服务器，RAID0 RAID5

我们会根据数据的存储和访问的需求，去匹配对应的RAID级别

#### 5、LVS、Nginx、HAproxy有什么区别？工作中你怎么选择？

[LVS](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247493540&idx=1&sn=674ab34fa6064f8d2bc39b1c1a6f2413&chksm=9ac6a42eadb12d38622b494d822e6c3b070edfde039b4900304f4b2f0ad452f9abe40b4a5c63&scene=21#wechat_redirect)：是基于四层的转发
[HAproxy](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247513791&idx=1&sn=1e5480f431a24d031c4f7c32a4dbf52b&chksm=9ac6f535adb17c236a6002e4c9bb2ad8bb149ff2c43e97f51b3c74973f1647c77ef5f629fc33&scene=21#wechat_redirect)：是基于四层和七层的转发，是专业的代理服务器
[Nginx](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488951&idx=1&sn=8b914c766d077af05ecf114ea14bb521&chksm=9ac5563dadb2df2b4e9e6628dbd038365dd836c7bfeb01676df30cd0c0589d1786ad2417cc07&scene=21#wechat_redirect)：是WEB服务器，缓存服务器，又是反向代理服务器，可以做七层的转发

区别：LVS由于是基于四层的转发所以只能做端口的转发、而基于URL的、基于目录的这种转发LVS就做不了

工作选择：HAproxy和Nginx由于可以做七层的转发，所以URL和目录的转发都可以做，在很大并发量的时候我们就要选择LVS，像中小型公司的话并发量没那么大，选择HAproxy或者Nginx足已，由于HAproxy由是专业的代理服务器，配置简单，所以中小型企业推荐使用HAproxy

#### 6、Squid、Varinsh和Nginx有什么区别，工作中你怎么选择？

[Squid](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488666&idx=1&sn=dee1eed06f1df15429f97638ffe15dee&chksm=9ac55710adb2de06336eb59d35beb59261aa7e99a492cdafad82f573261cb8dd28accf169600&scene=21#wechat_redirect)、Varinsh和[Nginx](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488906&idx=1&sn=d749aa4c1783f28ed3ee022b583a3b1c&chksm=9ac55600adb2df169ba93770c184b99fcfa69dd0eb42624c406fa0905df3cf3f8cdaab2cd0f8&scene=21#wechat_redirect)都是代理服务器

###### 什么是代理服务器：

能代替用户去访问公网，并且能把访问到的数据缓存到服务器本地，等用户下次再访问相同的资源的时候，代理服务器直接从本地回应给用户，当本地没有的时候，我代替你去访问公网，我接收你的请求，我先在我自已的本地缓存找，如果我本地缓存有，我直接从我本地的缓存里回复你如果我在我本地没有找到你要访问的缓存的数据，那么代理服务器就会代替你去访问公网

###### 区别：

1）Nginx本来是反向代理/web服务器，用了插件可以做做这个副业
但是本身不支持特性挺多，只能缓存静态文件

2）从这些功能上。varnish和squid是专业的cache服务，而nginx这些是第三方模块完成

3）varnish本身的技术上优势要高于squid，它采用了可视化页面缓存技术
在内存的利用上，Varnish比Squid具有优势，性能要比Squid高。
还有强大的通过Varnish管理端口，可以使用正则表达式快速、批量地清除部分缓存
它是内存缓存，速度一流，但是内存缓存也限制了其容量，缓存页面和图片一般是挺好的

4）[squid](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488707&idx=1&sn=31d24eef449a10f3ada90828b6b417b5&chksm=9ac55749adb2de5f314e1543610b508fabf6eb74f366f7c6dd859d3a94376606e40a44f82e12&scene=21#wechat_redirect)的优势在于完整的庞大的cache技术资料，和很多的应用生产环境

###### 工作中选择：

要做cache服务的话，我们肯定是要选择专业的cache服务，优先选择squid或者varnish。

##### 7、Tomcat和Resin有什么区别，工作中你怎么选择？

区别：[Tomcat](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247491438&idx=1&sn=f118f04d3fa6ae31c996ea978811e366&chksm=9ac55ce4adb2d5f2b440a6b2ea5e8300885e4c5b01e9afe6d41dfd837fcdc2e4646a9aadc989&scene=21#wechat_redirect)用户数多，可参考文档多，Resin用户数少，可考虑文档少，最主要区别则是Tomcat是标准的java容器，不过性能方面比resin的要差一些，但稳定性和java程序的兼容性，应该是比resin的要好

工作中选择：现在大公司都是用resin，追求性能；而中小型公司都是用Tomcat，追求稳定和程序的兼容

#### 8、什么是中间件？什么是jdk？

中间件是一种独立的系统软件或服务程序，分布式应用软件借助这种软件在不同的技术之间共享资源

中间件位于客户机/ 服务器的操作系统之上，管理计算机资源和网络通讯是连接两个独立应用程序或独立系统的软件。相连接的系统，即使它们具有不同的接口

但通过中间件相互之间仍能交换信息。执行中间件的一个关键途径是信息传递通过中间件，应用程序可以工作于多平台或OS环境。

jdk：jdk是Java的开发工具包，它是一种用于构建在 Java 平台上发布的应用程序、applet 和组件的开发环境

#### 9、讲述一下Tomcat8005、8009、8080三个端口的含义？

8005==》 关闭时使用
8009==》 为AJP端口，即容器使用，如Apache能通过AJP协议访问Tomcat的8009端口
8080==》 一般应用使用

#### 10、什么叫CDN？

即内容分发网络，其目的是通过在现有的Internet中增加一层新的网络架构，将网站的内容发布到最接近用户的网络边缘，使用户可就近取得所需的内容，提高用户访问网站的速度。

#### 11、什么叫网站灰度发布？

灰度发布是指在黑与白之间，能够平滑过渡的一种发布方式
AB test就是一种灰度发布方式，让一部用户继续用A，一部分用户开始用B
如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面 来
灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度

#### 12、简述DNS进行域名解析的过程？

用户要访问www.baidu.com，会先找本机的host文件，再找本地设置的[DNS](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247487258&idx=1&sn=0edc549cde58680a6afb6d684f8779b0&chksm=9ac54c90adb2c58623db6f5025af78655dd1837d0526b84300f154463129a6c0314e68d053fc&scene=21#wechat_redirect)服务器，如果也没有的话，就去网络中找根服务器，根服务器反馈结果，说只能提供一级域名服务器.cn，就去找一级域名服务器，一级域名服务器说只能提供二级域名服务器.com.cn,就去找二级域名服务器，二级域服务器只能提供三级域名服务器.baidu.com.cn，就去找三级域名服务器，三级域名服务器正好有这个网站www.baidu.com，然后发给请求的服务器，保存一份之后，再发给客户端

#### 13、RabbitMQ是什么东西？

RabbitMQ也就是消息队列中间件，消息中间件是在消息的传息过程中保存消息的容器
消息中间件再将消息从它的源中到它的目标中标时充当中间人的作用
队列的主要目的是提供路由并保证消息的传递；如果发送消息时接收者不可用
消息队列不会保留消息，直到可以成功地传递为止，当然，消息队列保存消息也是有期限地

#### 14、讲一下Keepalived的工作原理？

在一个虚拟路由器中，只有作为MASTER的VRRP路由器会一直发送VRRP通告信息,
BACKUP不会抢占MASTER，除非它的优先级更高。当MASTER不可用时(BACKUP收不到通告信息)
多台BACKUP中优先级最高的这台会被抢占为MASTER。这种抢占是非常快速的(<1s)，以保证服务的连续性
由于安全性考虑，VRRP包使用了加密协议进行加密。BACKUP不会发送通告信息，只会接收通告信息

#### 15、讲述一下LVS三种模式的工作过程？

[LVS](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247493540&idx=1&sn=674ab34fa6064f8d2bc39b1c1a6f2413&chksm=9ac6a42eadb12d38622b494d822e6c3b070edfde039b4900304f4b2f0ad452f9abe40b4a5c63&scene=21#wechat_redirect) 有三种负载均衡的模式，分别是VS/NAT（nat 模式） VS/DR(路由模式) VS/TUN（隧道模式）

###### 一、NAT模式（VS-NAT）

原理：就是把客户端发来的数据包的IP头的目的地址，在负载均衡器上换成其中一台RS的IP地址，并发至此RS来处理,RS处理完后把数据交给负载均衡器,负载均衡器再把数据包原IP地址改为自己的IP，将目的地址改为客户端IP地址即可期间,无论是进来的流量,还是出去的流量,都必须经过负载均衡器

优点：集群中的物理服务器可以使用任何支持[TCP/IP](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247503943&idx=1&sn=69870e2f574d7cefa741350d75d1fb60&chksm=9ac693cdadb11adb22c4b7dfa82101d7c4bfb8371e536d0d6256194a7ce1908e638da7a3c437&scene=21#wechat_redirect)操作系统，只有负载均衡器需要一个合法的IP地址

缺点：扩展性有限。当服务器节点（普通PC服务器）增长过多时,负载均衡器将成为整个系统的瓶颈，因为所有的请求包和应答包的流向都经过负载均衡器。当服务器节点过多时，大量的数据包都交汇在负载均衡器那，速度就会变慢！

###### 二、IP隧道模式（VS-TUN）

原理：首先要知道，互联网上的大多Internet服务的请求包很短小，而应答包通常很大，那么隧道模式就是，把客户端发来的数据包，封装一个新的IP头标记(仅目的IP)发给RS，RS收到后,先把数据包的头解开,还原数据包,处理后,直接返回给客户端,不需要再经过负载均衡器。注意,由于RS需要对负载均衡器发过来的数据包进行还原,所以说必须支持IPTUNNEL协议，所以,在RS的内核中,必须编译支持IPTUNNEL这个选项

优点：负载均衡器只负责将请求包分发给后端节点服务器，而RS将应答包直接发给用户，所以，减少了负载均衡器的大量数据流动，负载均衡器不再是系统的瓶颈，就能处理很巨大的请求量，这种方式，一台负载均衡器能够为很多RS进行分发。而且跑在公网上就能进行不同地域的分发。

缺点：隧道模式的RS节点需要合法IP，这种方式需要所有的服务器支持”IP Tunneling”(IP Encapsulation)协议，服务器可能只局限在部分Linux系统上

###### 三、直接路由模式（VS-DR）

原理：负载均衡器和RS都使用同一个IP对外服务但只有DR对ARP请求进行响应，所有RS对本身这个IP的ARP请求保持静默也就是说,网关会把对这个服务IP的请求全部定向给DR，而DR收到数据包后根据调度算法,找出对应的RS,把目的MAC地址改为RS的MAC（因为IP一致），并将请求分发给这台RS这时RS收到这个数据包,处理完成之后，由于IP一致，可以直接将数据返给客户，则等于直接从客户端收到这个数据包无异,处理后直接返回给客户端，由于负载均衡器要对二层包头进行改换,所以负载均衡器和RS之间必须在一个广播域，也可以简单的理解为在同一台交换机上

优点：和TUN（隧道模式）一样，负载均衡器也只是分发请求，应答包通过单独的路由方法返回给客户端，与VS-TUN相比，VS-DR这种实现方式不需要隧道结构，因此可以使用大多数操作系统做为物理服务器。

缺点：（不能说缺点，只能说是不足）要求负载均衡器的网卡必须与物理网卡在一个物理段上。

#### 16、mysql的innodb如何定位锁问题，mysql如何减少主从复制延迟？

mysql的innodb如何定位锁问题:
在使用 show engine innodb status检查引擎状态时，发现了死锁问题
在5.5中，information_schema 库中增加了三个关于锁的表（MEMORY引擎）

```
innodb_trx         ## 当前运行的所有事务

innodb_locks     ## 当前出现的锁

innodb_lock_waits  ## 锁等待的对应关系
```

[mysql](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247514815&idx=1&sn=fa563c45ccc54c0bd9a93ae2f2abbfa5&chksm=9ac6f935adb1702361fe9f354ac2dd2ee3b1d9a8a0839b9a0e106050ccc7d6b8ff6d119b2d33&scene=21#wechat_redirect)如何减少主从复制延迟:
如果延迟比较大，就先确认以下几个因素：
1.从库硬件比主库差，导致复制延迟
2.主从复制单线程，如果主库写并发太大，来不及传送到从库就会导致延迟。更高版本的mysql可以支持多线程复制
3.慢SQL语句过多
4.网络延迟
5.master负载：主库读写压力大，导致复制延迟，架构的前端要加buffer及缓存层
6.slave负载：一般的做法是，使用多台slave来分摊读请求，再从这些slave中取一台专用的服务器

只作为备份用，不进行其他任何操作.另外， 2个可以减少延迟的参数:

```
–slave-net-timeout=seconds 单位为秒 默认设置为 3600秒

#参数含义：当slave从主数据库读取log数据失败后，等待多久重新建立连接并获取数据

–master-connect-retry=seconds 单位为秒 默认设置为 60秒

#参数含义：当重新建立主从连接时，如果连接建立失败，间隔多久后重试
```

通常配置以上2个参数可以减少网络问题导致的主从数据同步延迟

[MySQL](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247487019&idx=1&sn=f0117cd6772af05f25d12f1716492ff5&chksm=9ac54da1adb2c4b79a792393534e62d4494da2be648c4bf3d0845c2448d4d6da124c13e51886&scene=21#wechat_redirect)数据库主从同步延迟解决方案

最简单的减少slave同步延时的方案就是在架构上做优化，尽量让主库的DDL快速执行

还有就是主库是写，对数据安全性较高，比如sync_binlog=1，innodb_flush_log_at_trx_commit
= 1 之类的设置，而slave则不需要这么高的数据安全，完全可以讲sync_binlog设置为0或者关闭binlog

innodb_flushlog也可以设置为0来提高sql的执行效率。另外就是使用比主库更好的硬件设备作为slave

#### 17、如何重置mysql root密码？

一、 在已知[MYSQL](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247487014&idx=2&sn=d2406f1980fd33d3412fd828e2ee4d2a&chksm=9ac54dacadb2c4ba925cbb0aadd7f42f156dde9644df96aaeefdc2184735f2686386dcbb277f&scene=21#wechat_redirect)数据库的ROOT用户密码的情况下，修改密码的方法：

在[SHELL](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247512842&idx=1&sn=7898565bd73db5c0eb6d58145d2fa21a&chksm=9ac6f080adb179964fee01fe70dff1fe9d0b6426a0d2c11c86ed461bef87875fba7603187cb7&scene=21#wechat_redirect)环境下，使用mysqladmin命令设置：

```
mysqladmin –u root –p password “新密码”   回车后要求输入旧密码
```

在mysql>环境中,使用update命令，直接更新mysql库user表的数据：

```
Update  mysql.user  set  password=password(‘新密码’)  where  user=’root’;

flush   privileges;
```

注意：mysql语句要以分号”；”结束

在mysql>环境中，使用grant命令，修改root用户的授权权限。

```
grant  all  on  *.*  to   root@’localhost’  identified  by  ‘新密码’；
```

二、 如忘记了[mysql](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247487014&idx=2&sn=d2406f1980fd33d3412fd828e2ee4d2a&chksm=9ac54dacadb2c4ba925cbb0aadd7f42f156dde9644df96aaeefdc2184735f2686386dcbb277f&scene=21#wechat_redirect)数据库的ROOT用户的密码，又如何做呢？方法如下：

关闭当前运行的mysqld服务程序：service  mysqld  stop（要先将mysqld添加为系统服务）

使用mysqld_safe脚本以安全模式（不加载授权表）启动mysqld 服务

```
/usr/local/mysql/bin/mysqld_safe  --skip-grant-table  &
```

使用空密码的root用户登录数据库，重新设置ROOT用户的密码

```
＃mysql  -u   root

Mysql> Update  mysql.user  set  password=password(‘新密码’)  where  user=’root’;

Mysql> flush   privileges;
```

#### 18、lvs/nginx/haproxy优缺点

##### Nginx的优点是：

1、工作在网络的7层之上，可以针对http应用做一些分流的策略，比如针对域名、目录结构

它的正则规则比HAProxy更为强大和灵活，这也是它目前广泛流行的主要原因之一

[Nginx](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247509132&idx=1&sn=db184e77db62ac22ecb56f852d7964bc&chksm=9ac6e706adb16e103fb303f13367540486eaab247be6aacbcbf202436044eecfcb9847e74f76&scene=21#wechat_redirect)单凭这点可利用的场合就远多于LVS了。

2、[Nginx](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247515472&idx=2&sn=1b5ace0209a88bbbcd723bd230bf6cc8&chksm=9ac6fedaadb177cc086dd051f56cca26bc243ed76adc70061e9829c4f84e0ef492605f2d1117&scene=21#wechat_redirect)对网络稳定性的依赖非常小，理论上能ping通就就能进行负载功能，这个也是它的优势之一

相反[LVS](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247493540&idx=1&sn=674ab34fa6064f8d2bc39b1c1a6f2413&chksm=9ac6a42eadb12d38622b494d822e6c3b070edfde039b4900304f4b2f0ad452f9abe40b4a5c63&scene=21#wechat_redirect)对网络稳定性依赖比较大，这点本人深有体会；

3、[Nginx安装](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488784&idx=2&sn=03646237396703c418d4f262ced56041&chksm=9ac5569aadb2df8c0b670bdfb0e55aa51374272a4e5f97d84bcfdc8db0606df4252a0dd14708&scene=21#wechat_redirect)和配置比较简单，测试起来比较方便，它基本能把错误用日志打印出来

[LVS](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247493540&idx=1&sn=674ab34fa6064f8d2bc39b1c1a6f2413&chksm=9ac6a42eadb12d38622b494d822e6c3b070edfde039b4900304f4b2f0ad452f9abe40b4a5c63&scene=21#wechat_redirect)的配置、测试就要花比较长的时间了，LVS对网络依赖比较大。

4、可以承担高负载压力且稳定，在硬件不差的情况下一般能支撑几万次的并发量，负载度比LVS相对小些。

5、Nginx可以通过端口检测到服务器内部的故障，比如根据服务器处理网页返回的状态码、超时等等，并且会把返回错误的请求重新提交到另一个节点，不过其中缺点就是不支持url来检测。比如用户正在上传一个文件，而处理该上传的节点刚好在上传过程中出现故障，Nginx会把上传切到另一台服务器重新处理，而LVS就直接断掉了

如果是上传一个很大的文件或者很重要的文件的话，用户可能会因此而不满。

6、Nginx不仅仅是一款优秀的负载均衡器/反向代理软件，它同时也是功能强大的Web应用服务器

[LNMP](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247489128&idx=1&sn=222f79e4947ef886a26acdeb05506928&chksm=9ac555e2adb2dcf4ca8ca449e1f340dfdc15d841a530e632c346ad5a61f0da297e785ae3882f&scene=21#wechat_redirect)也是近几年非常流行的web架构，在高流量的环境中稳定性也很好。

7、Nginx现在作为Web反向加速缓存越来越成熟了，速度比传统的[Squid服务器](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488666&idx=1&sn=dee1eed06f1df15429f97638ffe15dee&chksm=9ac55710adb2de06336eb59d35beb59261aa7e99a492cdafad82f573261cb8dd28accf169600&scene=21#wechat_redirect)更快，可考虑用其作为反向代理加速器

8、Nginx可作为中层[反向代理](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247488798&idx=1&sn=aa647457c6aecab6ac08d6bcbab7f9b3&chksm=9ac55694adb2df82d263a96bf844c6eb34011743103baa48ff9f3cb2847e03de49bf11f864ce&scene=21#wechat_redirect)使用，这一层面Nginx基本上无对手，唯一可以对比Nginx的就只有lighttpd了

不过lighttpd目前还没有做到Nginx完全的功能，配置也不那么清晰易读，社区资料也远远没Nginx活跃

9、Nginx也可作为静态网页和图片服务器，这方面的性能也无对手。还有Nginx社区非常活跃，第三方模块也很多

##### Nginx的缺点是：

1、Nginx仅能支持http、https和Email协议，这样就在适用范围上面小些，这个是它的缺点

2、对后端服务器的健康检查，只支持通过端口来检测，不支持通过url来检测

不支持Session的直接保持，但能通过ip_hash来解决

LVS：使用Linux内核集群实现一个高性能、高可用的负载均衡服务器

它具有很好的可伸缩性（Scalability)、可靠性（Reliability)和可管理性（Manageability)

##### LVS的优点是：

1、抗负载能力强、是工作在网络4层之上仅作分发之用，没有流量的产生

这个特点也决定了它在负载均衡软件里的性能最强的，对内存和cpu资源消耗比较低

2、配置性比较低，这是一个缺点也是一个优点，因为没有可太多配置的东西

所以并不需要太多接触，大大减少了人为出错的几率

3、工作稳定，因为其本身抗负载能力很强，自身有完整的双机热备方案

如`LVS+Keepalived`，不过我们在项目实施中用得最多的还是LVS/DR+Keepalived

4、无流量，LVS只分发请求，而流量并不从它本身出去，这点保证了均衡器IO的性能不会收到大流量的影响。

5、应用范围较广，因为LVS工作在4层，所以它几乎可对所有应用做负载均衡，包括http、数据库、在线聊天室等

##### LVS的缺点是：

1、软件本身不支持正则表达式处理，不能做动静分离

而现在许多网站在这方面都有较强的需求，这个是Nginx/HAProxy+Keepalived的优势所在

2、如果是网站应用比较庞大的话，LVS/DR+Keepalived实施起来就比较复杂了

特别后面有Windows Server的机器的话，如果实施及配置还有维护过程就比较复杂了

相对而言，Nginx/HAProxy+Keepalived就简单多了。

##### HAProxy的特点是：

1、HAProxy也是支持虚拟主机的。

2、HAProxy的优点能够补充Nginx的一些缺点，比如支持Session的保持，Cookie的引导

同时支持通过获取指定的url来检测后端服务器的状态

3、HAProxy跟LVS类似，本身就只是一款负载均衡软件

单纯从效率上来讲HAProxy会比Nginx有更出色的负载均衡速度，在并发处理上也是优于Nginx的

4、HAProxy支持TCP协议的负载均衡转发，可以对MySQL读进行负载均衡

对后端的MySQL节点进行检测和负载均衡，大家可以用LVS+Keepalived对MySQL主从做负载均衡

5、HAProxy负载均衡策略非常多，HAProxy的负载均衡算法现在具体有如下8种：

- roundrobin，表示简单的轮询，这个不多说，这个是负载均衡基本都具备的；
- static-rr，表示根据权重，建议关注；
- leastconn，表示最少连接者先处理，建议关注；
- source，表示根据请求源IP，这个跟Nginx的IP_hash机制类似,我们用其作为解决session问题的一种方法，建议关注；
- ri，表示根据请求的URI；
- rl_param，表示根据请求的URl参数’balance url_param’ requires an URL parameter name；
- hdr(name)，表示根据HTTP请求头来锁定每一次HTTP请求；
- rdp-cookie(name)，表示根据据cookie(name)来锁定并哈希每一次TCP请求。

#### 19、mysql数据备份工具

##### mysqldump工具

mysqldump是mysql自带的备份工具，目录在bin目录下面：/usr/local/mysql/bin/mysqldump。支持基于innodb的热备份，但是由于是逻辑备份，所以速度不是很快，适合备份数据比较小的场景，Mysqldump完全备份+二进制日志可以实现基于时间点的恢复。

##### 基于LVM快照备份

在物理备份中，有基于文件系统的物理备份（LVM的快照），也可以直接用tar之类的命令对整个数据库目录，进行打包备份，但是这些只能进行冷备份，不同的存储引擎备份的也不一样，myisam自动备份到表级别，而innodb不开启独立表空间的话只能备份整个数据库。

##### tar包备份

percona提供的xtrabackup工具，支持innodb的物理热备份，支持完全备份，增量备份，而且速度非常快，支持innodb存储引起的数据在不同，数据库之间迁移，支持复制模式下的从机备份恢复备份恢复，为了让xtrabackup支持更多的功能扩展，可以设立独立表空间，打开 innodb_file_per_table功能，启用之后可以支持单独的表备份

#### 20、keepalive的工作原理和如何做到健康检查

keepalived是以VRRP协议为实现基础的，VRRP全称Virtual Router Redundancy Protocol，即虚拟路由冗余协议。虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务的vip（该路由器所在局域网内，其他机器的默认路由为该vip），master会发组播，当backup收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个backup当master。这样就可以保证路由器的高可用了

keepalived主要有三个模块，分别是core、check和vrrp。core模块为keepalived的核心，负责主进程的启动、维护及全局配置文件的加载和解析。check负责健康检查，包括常见的各种检查方式，vrrp模块是来实现VRRP协议的

Keepalived健康检查方式配置

```
HTTP_GET|SSL_GET
HTTP_GET | SSL_GET
{
url {
path /# HTTP/SSL 检查的url可以是多个
digest <STRING> # HTTP/SSL 检查后的摘要信息用工具genhash生成
status_code 200# HTTP/SSL 检查返回的状态码
}
connect_port 80 # 连接端口
bindto<IPADD>
connect_timeout 3 # 连接超时时间
nb_get_retry 3 # 重连次数
delay_before_retry 2 #连接间隔时间
}
```

#### 21、统计ip访问情况，要求分析nginx访问日志，找出访问页面数量在前十位的ip

```
cat access.log | awk '{print $1}' | uniq -c | sort -rn | head -10
```

#### 22、使用tcpdump监听主机为192.168.1.1，tcp端口为80的数据，同时将输出结果保存输出到tcpdump.log

```
tcpdump 'host 192.168.1.1 and port 80' > tcpdump.log
```

#### 23、如何将本地80 端口的请求转发到8080 端口，当前主机IP 为192.168.2.1

```
iptables -A PREROUTING -d 192.168.2.1 -p tcp -m tcp -dport 80 -j DNAT-to-destination 192.168.2.1:8080
```

#### 24、简述raid0 raid1 raid5 三种工作模式的工作原理及特点

RAID 0：带区卷，连续以位或字节为单位分割数据，并行读/写于多个磁盘上，因此具有很高的数据传输率，但它没有数据冗余，RAID 0 只是单纯地提高性能，并没有为数据的可靠性提供保证，而且其中的一个磁盘失效将影响到所有数据。因此，RAID 0 不能应用于数据安全性要求高的场合

RAID 1：镜像卷，它是通过磁盘数据镜像实现数据冗余，在成对的独立磁盘上产生互为备份的数据，不能提升写数据效率。当原始数据繁忙时，可直接从镜像拷贝中读取数据，因此RAID1 可以提高读取性能，RAID 1 是磁盘阵列中单位成本最高的，镜像卷可用容量为总容量的1/2，但提供了很高的数据安全性和可用性，当一个磁盘失效时，系统可以自动切换到镜像磁盘上读写，而不需要重组失效的数据

RAID5：至少由3块硬盘组成，分布式奇偶校验的独立磁盘结构，它的奇偶校验码存在于所有磁盘上，任何一个硬盘损坏，都可以根据其它硬盘上的校验位来重建损坏的数据（最多允许1块硬盘损坏），所以raid5可以实现数据冗余，确保数据的安全性，同时raid5也可以提升数据的读写性能

#### 25、你对现在运维工程师的理解和以及对其工作的认识

运维工程师在公司当中责任重大，需要保证时刻为公司及客户提供最高、最快、最稳定、最安全的服务

运维工程师的一个小小的失误，很有可能会对公司及客户造成重大损失

因此运维工程师的工作需要严谨及富有创新精神

#### 26、实时抓取并显示当前系统中tcp 80端口的网络数据信息，请写出完整操作命令

```
tcpdump -nn tcp port 80
```

#### 27、服务器开不了机怎么解决一步步的排查

A、造成服务器故障的原因可能有以下几点：

![图片](https://mmbiz.qpic.cn/mmbiz_png/nDMNE6lrvW7QAOEibicicuXqDoe4jibUm0ExAjpvfSFuNjYGe2ONVbYzvTrnb8ckjNdRjLPlBphZMCfNmB4IZzDAhg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

B、如何排查服务器故障的处理步骤如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/nDMNE6lrvW7QAOEibicicuXqDoe4jibUm0ExGUQw7R6ycDpHpkcG6LyYYiac7yoHdKaticvFchQ0ibtKCu8Pdc5Fuewgw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 28、Linux系统中病毒怎么解决

1）最简单有效的方法就是重装系统

2）要查的话就是找到病毒文件然后删除，中毒之后一般机器cpu、内存使用率会比较高，机器向外发包等异常情况，排查方法简单介绍下

- top 命令找到cpu使用率最高的进程
- 一般病毒文件命名都比较乱，可以用 ps aux 找到病毒文件位置
- rm -f  命令删除病毒文件
- 检查计划任务、开机启动项和病毒文件目录有无其他可以文件等

3）由于即使删除病毒文件不排除有潜伏病毒，所以最好是把机器备份数据之后重装一下

#### 29、发现一个病毒文件你删了他又自动创建怎么解决

公司的内网某台linux服务器流量莫名其妙的剧增,用iftop查看有连接外网的情况，针对这种情况一般重点查看netstat连接的外网ip和端口。

用lsof -p pid可以查看到具体是那些进程，哪些文件经查勘发现/root下有相关的配置conf.n hhe两个可疑文件，rm -rf后不到一分钟就自动生成了，由此推断是某个母进程产生的这些文件。所以找到母进程就是找到罪魁祸首

查杀病毒最好断掉外网访问，还好是内网服务器，可以通过内网访问，断了内网，病毒就失去外联的能力，杀掉它就容易的多，怎么找到呢，找了半天也没有看到蛛丝马迹，没办法只有ps axu一个个排查，方法是查看可以的用户和和系统相似而又不是的冒牌货，果然，看到了如下进程可疑

看不到图片就是/usr/bin/.sshd，于是我杀掉所有.sshd相关的进程，然后直接删掉.sshd这个可执行文件，然后才删掉了文章开头提到的自动复活的文件

总结一下，遇到这种问题，如果不是太严重，尽量不要重装系统

一般就是先断外网，然后利用`iftop`,`ps`,`netstat`,`chattr`,`lsof`,`pstree`这些工具顺藤摸瓜

一般都能找到元凶。但是如果遇到诸如此类的问题

`/boot/efi/EFI/redhat/grub.efi: Heuristics.Broken.Executable FOUND`，个人觉得就要重装系统了

#### 30、说说TCP/IP的七层模型

应用层 (Application)：
网络服务与最终用户的一个接口。
协议有：HTTP FTP TFTP SMTP SNMP DNS TELNET HTTPS POP3 DHCP

表示层（Presentation Layer）：
数据的表示、安全、压缩。（在五层模型里面已经合并到了应用层）
格式有，JPEG、ASCll、DECOIC、加密格式等

会话层（Session Layer）：
建立、管理、终止会话。（在五层模型里面已经合并到了应用层）
对应主机进程，指本地主机与远程主机正在进行的会话

传输层 (Transport)：
定义传输数据的协议端口号，以及流控和差错校验。
协议有：TCP UDP，数据包一旦离开网卡即进入网络传输层

网络层 (Network)：
进行逻辑地址寻址，实现不同网络之间的路径选择。
协议有：ICMP IGMP IP（IPV4 IPV6） ARP RARP

数据链路层 (Link)：
建立逻辑连接、进行硬件地址寻址、差错校验等功能。（由底层网络定义协议）
将比特组合成字节进而组合成帧，用MAC地址访问介质，错误发现但不能纠正

物理层（Physical Layer）：
是计算机网络OSI模型中最低的一层

物理层规定:为传输数据所需要的物理链路创建、维持、拆除而提供具有机械的，电子的，功能的和规范的特性

简单的说，物理层确保原始的数据可在各种物理媒体上传输。局域网与广域网皆属第1、2层

物理层是OSI的第一层，它虽然处于最底层，却是整个开放系统的基础

物理层为设备之间的数据通信提供传输媒体及互连设备，为数据传输提供可靠的环境

如果您想要用尽量少的词来记住这个第一层，那就是“信号和介质”

#### 31、你常用的Nginx模块，用来做什么

> rewrite模块，实现重写功能
> access模块：来源控制
> ssl模块：安全加密
> ngx_http_gzip_module：网络传输压缩模块
> ngx_http_proxy_module 模块实现代理
> ngx_http_upstream_module模块实现定义后端服务器列表
> ngx_cache_purge实现缓存清除功能

#### 32、请列出你了解的web服务器负载架构

> Nginx 
> Haproxy
> Keepalived
> LVS 

#### 33、查看http的并发请求数与其TCP连接状态

```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'


还有ulimit -n 查看linux系统打开最大的文件描述符，这里默认1024

不修改这里web服务器修改再大也没用，若要用就修改很几个办法，这里说其中一个：

修改/etc/security/limits.conf
* soft nofile 10240
* hard nofile 10240
重启后生效
```

#### 34、用tcpdump嗅探80端口的访问看看谁最高

```
tcpdump -i eth0 -tnn dst port 80 -c 1000 | awk -F"." '{print $1"."$2"."$3"."$4}'| sort | uniq -c | sort -nr |head -20
```

#### 35、写一个脚本，实现判断192.168.1.0/24网络里，当前在线的IP有哪些，能ping通则认为在线

```
#!/bin/bash
for ip in `seq 1 255`
do

{

ping -c 1 192.168.1.$ip > /dev/null 2>&1
if [ $? -eq 0 ]; then
echo 192.168.1.$ip UP
else
echo 192.168.1.$ip DOWN
fi
}&
done
wait
```

#### 36、已知 apache 服务的访问日志按天记录在服务器本地目录/app/logs 下，由于磁盘空间紧张现在要求只能保留最近 7 天的访问日志！请问如何解决？请给出解决办法或配置或处理命令

```
创建文件脚本：

#!/bin/bash

for n in `seq 14`

do       

date -s "11/0$n/14"

touch access_www_`(date +%F)`.log

done



解决方法：

# pwd/application/logs

# ll

-rw-r--r--. 1 root root 0 Jan  1 00:00 access_www_2015-01-01.log
-rw-r--r--. 1 root root 0 Jan  2 00:00 access_www_2015-01-02.log
-rw-r--r--. 1 root root 0 Jan  3 00:00 access_www_2015-01-03.log
-rw-r--r--. 1 root root 0 Jan  4 00:00 access_www_2015-01-04.log
-rw-r--r--. 1 root root 0 Jan  5 00:00 access_www_2015-01-05.log
-rw-r--r--. 1 root root 0 Jan  6 00:00 access_www_2015-01-06.log
-rw-r--r--. 1 root root 0 Jan  7 00:00 access_www_2015-01-07.log
-rw-r--r--. 1 root root 0 Jan  8 00:00 access_www_2015-01-08.log
-rw-r--r--. 1 root root 0 Jan  9 00:00 access_www_2015-01-09.log
-rw-r--r--. 1 root root 0 Jan 10 00:00 access_www_2015-01-10.log
-rw-r--r--. 1 root root 0 Jan 11 00:00 access_www_2015-01-11.log
-rw-r--r--. 1 root root 0 Jan 12 00:00 access_www_2015-01-12.log
-rw-r--r--. 1 root root 0 Jan 13 00:00 access_www_2015-01-13.log

-rw-r--r--. 1 root root 0 Jan 14 00:00 access_www_2015-01-14.log

# find /application/logs/ -type f -mtime +7 -name "*.log"|xargs rm –f 

##也可以使用-exec rm -f {} \;进行删除

# ll

-rw-r--r--. 1 root root 0 Jan  7 00:00 access_www_2015-01-07.log
-rw-r--r--. 1 root root 0 Jan  8 00:00 access_www_2015-01-08.log
-rw-r--r--. 1 root root 0 Jan  9 00:00 access_www_2015-01-09.log
-rw-r--r--. 1 root root 0 Jan 10 00:00 access_www_2015-01-10.log
-rw-r--r--. 1 root root 0 Jan 11 00:00 access_www_2015-01-11.log
-rw-r--r--. 1 root root 0 Jan 12 00:00 access_www_2015-01-12.log
-rw-r--r--. 1 root root 0 Jan 13 00:00 access_www_2015-01-13.log

-rw-r--r--. 1 root root 0 Jan 14 00:00 access_www_2015-01-14.log
```

#### 37、如何优化 Linux系统（可以不说太具体）？

- 不用root，添加普通用户，通过sudo授权管理
- 更改默认的远程连接SSH服务端口及禁止root用户远程连接
- 定时自动更新服务器时间
- 配置国内yum源
- 关闭selinux及[iptables](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247486986&idx=1&sn=2a2eee9b49c765fc0f1878134d91352c&chksm=9ac54d80adb2c496e5cde96a182f8373429638629dfd14b5269660fdae2ed4aef1bdb16ed033&scene=21#wechat_redirect)（[iptables](http://mp.weixin.qq.com/s?__biz=MzAwMjg1NjY3Nw==&mid=2247487396&idx=1&sn=e6c5f2e9182fffd31212be2917fd19e0&chksm=9ac54c2eadb2c538e927b2ca0c49dcc062c411809aa05ba9c432f90145f5828d00657373dae9&scene=21#wechat_redirect)工作场景如果有外网IP一定要打开，高并发除外）
- 调整文件描述符的数量
- 精简开机启动服务（crond rsyslog network sshd）
- 内核参数优化（/etc/sysctl.conf）
- 更改字符集，支持中文，但建议还是用英文字符集，防止乱码
- 锁定关键系统文件
- 清空/etc/issue，去除系统及内核版本登录前的屏幕显示

#### 38、请执行命令取出 linux 中 eth0 的 IP 地址(请用 cut，有能力者也可分别用 awk,sed 命令答)

```shell
cut方法1：
# ifconfig eth0|sed -n '2p'|cut -d ":" -f2|cut -d " " -f1
192.168.20.130

awk方法2：
# ifconfig eth0|awk 'NR==2'|awk -F ":" '{print $2}'|awk '{print $1}'
192.168.20.130

awk多分隔符方法3：
# ifconfig eth0|awk 'NR==2'|awk -F "[: ]+" '{print $4}'
192.168.20.130

sed方法4：
# ifconfig eth0|sed -n '/inet addr/p'|sed -r 's#^.*ddr:(.*)Bc.*$#\1#g'
192.168.20.130
```

#### 39、请写出下面 linux SecureCRT 命令行快捷键命令的功能？

> Ctrl + a：光标移动到行首
> Ctrl + c：终止当前程序
> Ctrl + d：如果光标前有字符则删除，没有则退出当前中断
> Ctrl + e：光标移动到行尾
> Ctrl + l：清屏
> Ctrl + u：剪切光标以前的字符
> Ctrl + k：剪切光标以后的字符
> Ctrl + y：复制u/k的内容
> Ctrl + r：查找最近用过的命令
> tab：命令或路径补全
> Ctrl+shift+c：复制
> Ctrl+shift+v：粘贴

#### 40、每天晚上 12 点，打包站点目录/var/www/html 备份到/data 目录下（最好每次备份按时间生成不同的备份包）

```shell
# cat a.sh 
#/bin/bash
cd /var/www/ && /bin/tar zcf /data/html-`date +%m-%d%H`.tar.gz html/
# crontab –e
00 00 * * * /bin/sh /root/a.sh
```





