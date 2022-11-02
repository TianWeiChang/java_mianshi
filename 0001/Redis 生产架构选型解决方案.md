Redis 生产架构选型解决方案

在写开源项目的时候，想到了要支持多种`Redis`部署方式，于是对于这块的生产环境的架构选型展开调研。

## 引擎版本

推荐使用更新的引擎版本以支持更多的特性。

### Redis 6.0新特性说明

- 模块系统新增多个`API`。
- 支持`SSL/TLS`加密。
- 支持新的`Redis`协议：`RESP3`。
- 服务端支持多模式的客户端缓存。
- 支持多线程IO。
- 副本中支持无盘复制（`diskless replication`）。
- `Redis-benchmark`新增了`Redis`集群模式。
- 支持重写`Systemd`。
- 支持`Disque`模块。

### Redis 5.0新特性说明

- 云数据库`Redis 5.0`版本大幅度优化内核，运行更加稳定，同时新增Stream、账号管理、审计日志等多种特性，满足您更多场景下的使用需求。

- 新的数据类型：

  流数据（Stream）。

  详细说明请参见`Redis Streams`。

- 新增账号管理功能。

- 新增日志管理功能，支持审计日志、运行日志和慢日志，您可以通过日志管理查询读写操作、敏感操作（如KEYS、FLUSHALL）和管理类命令的使用记录以及慢日志。

- 新增基于快照的缓存分析功能。

- 新的定时器（Timers）、集群（ Cluster）和字典（Dictionary）模块的API。

- RDB中增加LFU和LRU信息。

- 集群管理器从Ruby （redis-trib.rb）移植到了redis-cli中的C语言代码。

- 新增有序集合（Sorted Set）命令ZPOPMIN、ZPOPMAX、BZPOPMIN和BZPOPMAX。

- 升级Active Defragmentation至v2版本。

- 增强HyperLogLog的实现。

- 优化内存统计报告。

- 为许多有子命令的命令增加了HELP子命令。

- 提高了客户端频繁连接和断开连接时的性能表现。

- 升级Jemalloc至5.1版本。

- 新增命令CLIENT ID和CLIENT UNBLOCK。

- 新增了为艺术而生的LOLWUT命令。

- 弃用slave术语（需要API向后兼容的情况例外）。

- 对网络层进行了多处优化。

- 进行了一些Lua相关的改进。

- 新增动态HZ（Dynamic HZ）以平衡空闲CPU使用率和响应性。

- 对Redis核心代码进行了重构并在许多方面进行了改进。

## 架构

您需要根据业务需求选择：

- **集群架构**可轻松突破Redis自身单线程瓶颈，满足大容量、高性能的业务需求。
- **主从架构**，提供高性能的缓存服务和数据高可靠。
- **读写分离架构**提供高可用、高性能、高灵活的读写分离服务，解决热点数据集中及高并发读取的业务需求，最大化地节约用户运维成本。

### 主从架构-双副本

采用主从（master-replica）模式搭建。主节点提供日常服务访问，备节点提供HA高可用，当主节点发生故障，系统会自动在30秒内切换至备节点，保证业务平稳运行。

#### **可靠性**

- 服务可靠采用双机主从（master-replica）架构，主从节点位于不同物理机。

  主节点对外提供访问，用户可通过Redis命令行和通用客户端进行数据的增删改查操作。

  当主节点出现故障，HA系统会自动进行主从切换，保证业务平稳运行。

- 数据可靠默认开启数据持久化功能，数据全部落盘。

  支持数据备份功能，用户可以针对备份集回滚实例或者克隆实例，有效地解决数据误操作等问题。

#### **使用场景**

- Redis作为持久化数据存储使用的业务标准版提供持久化机制及备份恢复机制，极大地保证数据可靠性。

- 单个Redis性能压力可控的业务由于Redis原生采用单线程机制，性能在10万QPS以下的业务建议使用。

  如果需要更高的性能要求，请选用集群版本。

- Redis命令相对简单，排序、计算类命令较少的业务由于Redis的单线程机制，CPU会成为主要瓶颈。

  如排序、计算类较多的业务建议选用集群版配置。

###  主从架构-单副本

可以在没有数据可靠性要求的纯缓存场景充分发挥性能优势。

#### **使用场景**

- 纯缓存类业务场景

单副本版本只有一个数据库节点，节点出现故障时，系统会重新拉起一个Redis进程（没有数据），当节点故障业务自动切换完成后，应用程序需要将数据重新预热，以免对后端数据库产生访问压力冲击。单副本架构不能提供数据可靠性，如果发生节点故障，您需要重新对业务进行预热，因此，在对数据可靠性要求较高的敏感性业务中，建议选用双副本架构。

- 单个Redis性能压力可控

由于Redis原生采用单线程机制，CPU为单核能力，性能在8万QPS的业务建议使用。如果需要更高的性能要求，请选用集群版配置。

- Redis命令相对简单，排序、计算类命令较少

由于Redis的单线程机制，CPU为主要瓶颈。如排序、计算类较多的业务建议选用集群版配置。

### 集群版-双副本

可轻松突破Redis自身单线程瓶颈，满足大容量、高性能的业务需求。双副本集群版实例采用集群架构，每个分片服务器采用主从（master-replica）双副本模式。**集群版支持代理和直连两种连接模式**，您可以根据本章节的说明，选择适合业务需求的连接模式。

### **代理模式**

**集群架构的本地盘实例默认采用代理（proxy）模式**，支持通过一个统一的连接地址（域名）访问Redis集群，客户端的请求通过代理服务器转发到各数据分片，代理服务器、数据分片和配置服务器均不提供单独的连接地址，降低了应用开发难度和代码复杂度。代理模式的服务架构图和组件说明如下。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQwvyXaUn4466asGTEnI3UBSLwaficmJPajQx5CXNPyJUon3N7IOvibBRwA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQwe3FtVnX5U9bnhLyCeN0adCLvIrs94fiaz9kAYhnD9JT5V275brlVricw/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### **直连模式**

因所有请求都要通过代理服务器转发，代理模式在降低业务开发难度的同时也会小幅度影响Redis服务的响应速度。如果业务对响应速度的要求非常高，您可以使用直连模式，绕过代理服务器直接连接后端数据分片，从而降低网络开销和服务响应时间。直连模式的服务架构和说明如下。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQwdEQtUic1Vib489xiajIo0jT4HAbOlnibm9ib79wNcsId9CqYojbMTddFO9w/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

前提条件 使用Jedis、PhpRedis等支持Redis Cluster的客户端。

- 使用不支持Redis Cluster的客户端，可能因客户端无法重定向请求到正确的分片而获取不到需要的数据。
- Jedis对于Redis Cluster的支持是基于JedisCluster这个类，详细说明请参见Jedis文档。
- 您可以在Redis官网的客户端列表里查找更多支持Redis Cluster的客户端。

使用自定义连接池的示例代码 

```java
import redis.clients.jedis.*;
import java.util.HashSet;
import java.util.Set; 

//欢迎关注公众号：松哥说编程
public class main {
    private static final int DEFAULT_TIMEOUT = 2000; 
    private static final int DEFAULT_REDIRECTIONS = 5;
    private static final JedisPoolConfig DEFAULT_CONFIG = new JedisPoolConfig();


    public static void main(String args[]){
        JedisPoolConfig config = new JedisPoolConfig();
        // 最大空闲连接数, 根据业务需要设置，不能超过实例规格规定的最大的连接数
        config.setMaxIdle(200);
        // 最大连接数, 根据业务需要设置，不能超过实例规格规定的最大的连接数
        config.setMaxTotal(300);
        config.setTestOnBorrow(false);
        config.setTestOnReturn(false);

        // 开通直连访问时申请到的直连地址
        String host = "r-bp1xxxxxxxxxxxx.redis.rds.aliyuncs.com"; 
        int port = 6379;
        // 实例的密码
        String password = "xxxxx";

        Set<HostAndPort> jedisClusterNode = new HashSet<HostAndPort>();
        jedisClusterNode.add(new HostAndPort(host, port));
        JedisCluster jc = new JedisCluster(jedisClusterNode, DEFAULT_TIMEOUT, DEFAULT_TIMEOUT,
                DEFAULT_REDIRECTIONS,password, "clientName", config);
    }
}
```

### 集群版-单副本

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQwSV5uZ7v1tq0JDYcRno5PvDSDrveSBmWyj3WxS9mfK7ZDicSkRNcPRlQ/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQwdfXnibOjMHhNoRKy0859nWZNo93L2hbRsialne549Xiaib56sN2uPIj0Qg/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 读写分离版

针对读多写少的业务场景，提供高可用、高性能、灵活的读写分离服务，满足热点数据集中及高并发读取的业务需求，最大化地节约运维成本。读写分离版主要由主备节点、只读节点、Proxy（代理）节点和高可用系统组成。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQwB45Jib59aNZicR4xsswxSjXyIUWb569xEoPyrooW7YiaY0gJcRNaGeJ1w/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/sXiaukvjR0RDv7vocJgiaTnkickgnHH1YQw12XDUKhb7GLYAHW47dvvPKfqZBDryiabic2AFiacRGicfHMEL2ibDicAJia2A/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### **特点**

- **高可用**

1. 通过自研的高可用系统自动监控所有数据节点的健康状态，为整个实例的可用性保驾护航。

   主节点不可用时自动选择新的主节点并重新搭建复制拓扑。

   某个只读节点异常时，高可用系统能够自动探知并重新启动新节点完成数据同步，下线异常节点。

2. Proxy节点实时感知每个只读实例的服务状态。

   在某个只读实例异常期间，Proxy会自动降低该节点的服务权重，发现只读节点连续失败超过一定次数以后，会停止异常节点的服务权利，并具备继续监控后续重新启动节点服务的能力。

- **高性能**

1. 读写分离版采取链式复制架构，**可以通过扩展只读实例个数使整体实例性能呈线性增长**，同时基于源码层面对Redis复制流程的定制优化，可以最大程度地提升线性复制的系统稳定性，充分利用每一个只读节点的物理资源。

####　**使用场景**

读取请求｀QPS（Queries Per Second）｀压力较大

标准版Redis无法支撑较大的QPS，如果业务类型是读多写少类型，需要采用多个只读节点的部署方式来突破Redis单线程的性能瓶颈。Redis集群版提供1个、3个、5个只读节点的配置，相比标准版可以将QPS提升近5倍。

对Redis协议兼容性要求较高的业务 读写分离版完全兼容Redis协议命令，可将自建Redis数据库迁移至读写分离版，同时支持从Redis标准版（双副本）一键平滑升级至读写分离版。

####　**建议与使用须知**

- **当一个只读节点发生故障时，请求会转发到其他节点；如果所有只读节点均不可用，请求会全部转发到主节点。只读节点异常可能导致主节点负载提高、响应时间变长，因此在读负载高的业务场景建议使用多个只读节点。**
- **某些场景会触发只读节点的全量同步，例如在主节点触发高可用切换后。全量同步期间只读节点不提供服务并返回-LOADING Redis is loading the dataset in memory\r\n信息。**