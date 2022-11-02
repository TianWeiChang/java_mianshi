# Redis中主、从库宕机如何恢复？

### 1、什么是哨兵

哨兵是对Redis的系统的运行情况的监控，它是一个独立进程，功能有二个：

- 监控主数据库和从数据库是否运行正常；
- 主数据出现故障后自动将从数据库转化为主数据库；

### 2、原理

单个哨兵的架构：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1My8qiaxkd0icib9iboeRvHiaQ9NXJcu9GrZMEIojJrToKK43xJ0I9mdZh14Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

多个哨兵的架构：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1McS2ia98YEAO6SnxZGNySLvy816PP8QDfXrsjq5cpn9AmIib3IibiaZQmVQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

多个哨兵，不仅同时监控主从数据库，而且哨兵之间互为监控。

多个哨兵，防止哨兵单点故障。

### 3、环境

当前处于一主多从的环境中：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1M0L08CJ4oQWFM72sWTmWPCfdyReyXjPztibFX5Ky2GcicibgY7ib9kuJuOw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4、设置哨兵

启动哨兵进程首先需要创建哨兵配置文件：

```
vim sentinel.conf
```

输入内容：

```
sentinel monitor taotaoMaster 127.0.0.1 6379 1
```

说明：

- taotaoMaster：监控主数据的名称，自定义即可，可以使用大小写字母和“`.-_`”符号
- 127.0.0.1：监控的主数据库的IP
- 6379：监控的主数据库的端口
- 1：最低通过票数

启动哨兵进程：

```
redis-sentinel ./sentinel.conf
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1MPeqZm5Vf4Hztyls8vWjpIIJ8gSSwKGLNyflUFrsMZYQqYn1HibiaPCQQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由上图可以看到：

- 哨兵已经启动，它的id为9059917216012421e8e89a4aa02f15b75346d2b7
- 为master数据库添加了一个监控
- 发现了2个slave（由此可以看出，哨兵无需配置slave，只需要指定master，哨兵会自动发现slave）

### 5、从宕机及恢复

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1Myyb8chGaku95IqVM9SwoQh5nm17J5K9hU4SragjVibNszbtzXKq3bJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

kill掉2826进程后，30秒后哨兵的控制台输出：

```
2989:X 05 Jun 20:09:33.509 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
```

说明已经监控到slave宕机了，那么，如果我们将3380端口的redis实例启动后，会自动加入到主从复制吗？

```
2989:X 05 Jun 20:13:22.716 * +reboot slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379

2989:X 05 Jun 20:13:22.788 # -sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
```

可以看出，slave从新加入到了主从复制中。`-sdown：`说明是恢复服务。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1Mic52JcU824srRChibqbIfYWONGLF2oTFf200iaRvzIiax65HmnGC0h97kg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 6、主宕机及恢复

哨兵控制台打印出如下信息：

```
2989:X 05 Jun 20:16:50.300 # +sdown master taotaoMaster 127.0.0.1 6379  说明master服务已经宕机

2989:X 05 Jun 20:16:50.300 # +odown master taotaoMaster 127.0.0.1 6379 #quorum 1/1 

2989:X 05 Jun 20:16:50.300 # +new-epoch 1

2989:X 05 Jun 20:16:50.300 # +try-failover master taotaoMaster 127.0.0.1 6379  开始恢复故障

2989:X 05 Jun 20:16:50.304 # +vote-for-leader 9059917216012421e8e89a4aa02f15b75346d2b7 1  投票选举哨兵leader，现在就一个哨兵所以leader就自己

2989:X 05 Jun 20:16:50.304 # +elected-leader master taotaoMaster 127.0.0.1 6379  选中leader

2989:X 05 Jun 20:16:50.304 # +failover-state-select-slave master taotaoMaster 127.0.0.1 6379 选中其中的一个slave当做master

2989:X 05 Jun 20:16:50.357 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  选中6381

2989:X 05 Jun 20:16:50.357 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  发送slaveof no one命令

2989:X 05 Jun 20:16:50.420 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379   等待升级master

2989:X 05 Jun 20:16:50.515 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  升级6381为master

2989:X 05 Jun 20:16:50.515 # +failover-state-reconf-slaves master taotaoMaster 127.0.0.1 6379

2989:X 05 Jun 20:16:50.566 * +slave-reconf-sent slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379

2989:X 05 Jun 20:16:51.333 * +slave-reconf-inprog slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379

2989:X 05 Jun 20:16:52.382 * +slave-reconf-done slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379

2989:X 05 Jun 20:16:52.438 # +failover-end master taotaoMaster 127.0.0.1 6379 故障恢复完成

2989:X 05 Jun 20:16:52.438 # +switch-master taotaoMaster 127.0.0.1 6379 127.0.0.1 6381  主数据库从6379转变为6381

2989:X 05 Jun 20:16:52.438 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6381  添加6380为6381的从库

2989:X 05 Jun 20:16:52.438 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  添加6379为6381的从库

2989:X 05 Jun 20:17:22.463 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381 发现6379已经宕机，等待6379的恢复
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1M3ZI0CspkVFzGULFqdyZxTuZbiamELjNHLSlpwY7NuhBepibA8ZCYYTJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看出，目前，6381位master，拥有一个slave为6380.

接下来，我们恢复6379查看状态：

```
2989:X 05 Jun 20:35:32.172 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  6379已经恢复服务

2989:X 05 Jun 20:35:42.137 * +convert-to-slave slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  将6379设置为6381的slave
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1M194qbwzKpM0sYI3CG9abseb8dwjeGQpoCF1TeibLKibdmas4jMtz1JfQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 7、配置多个哨兵

```
vim sentinel.conf
```

输入内容：

```
sentinel monitor taotaoMaster1 127.0.0.1 6381 1

sentinel monitor taotaoMaster2 127.0.0.1 6381 2
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbue05yD3TJNUhgfo32pCfE1M2wLpXVY5D9sRhZW8sL6fwc9T3ZvPMyyUdo0Zjkk2hNns6CZS4wDI5A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

 dadadd