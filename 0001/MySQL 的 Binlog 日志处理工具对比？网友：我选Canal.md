MySQL 的 Binlog 日志处理工具对比？网友：我选Canal...

## **Canal** 

定位：基于数据库增量日志解析，提供增量数据订阅&消费，目前主要支持了`MySQL`。 

- 原理：

  - canal模拟`MySQL slave`的交互协议，伪装自己为`MySQLslave`，向`mysql master`发送`dump`协议
  - `MySQL master`收到dump请求，开始推送`binary log`给slave(也就是canal)
  - canal解析`binary log`对象(原始为byte流)

  ![canal架构图.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQxMzUwLWZhYzMwYmQ3NGZhMjA0ZDUucG5n?x-oss-process=image/format,png) 

  ![EventParser架构.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQxMzUwLTJjNjgzOGViN2NhOTcxNWMucG5n?x-oss-process=image/format,png) 

  整个parser过程大致可分为几步：

  - `Connection`获取上一次解析成功的位置（如果第一次启动，则获取初始制定的位置或者是当前数据库的binlog位点）
  - `Connection`建立连接，发生`BINLOG_DUMP`命令
  - `MySQL`开始推送`Binary Log`
  - 接收到的`Binary Log`通过`Binlog parser`进行协议解析，补充一些特定信息
  - 传递给`EventSink`模块进行数据存储，是一个阻塞操作，直到存储成功
  - 存储成功后，定时记录Binary Log位置

  ![Sink.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQxMzUwLThmMjhmMzE1ZDA0ZjVlYjgucG5n?x-oss-process=image/format,png) 

  - 数据过滤：支持通配符的过滤模式，表名，字段内容等
  - 数据路由/分发：解决1:n (1个parser对应多个store的模式)
  - 数据归并：解决n:1 (多个parser对应1个store)
  - 数据加工：在进入store之前进行额外的处理，比如join

  ## Maxwell

  ![canal、maxwell、mysql_streamer对比](https://imgconvert.csdnimg.cn/aHR0cDovL2ltYWdlLmxhaWppYW5mZW5nLm9yZy8yMDE5MDMxMF8xNjMwMjMucG5n?x-oss-process=image/format,png) 

  canal 由Java开发，分为服务端和客户端，拥有众多的衍生应用，性能稳定，功能强大；canal 需要自己编写客户端来消费canal解析到的数据。

  maxwell相对于canal的优势是使用简单，它直接将数据变更输出为json字符串，不需要再编写客户端。

  ## Databus

  `Databus`是一种低延迟变化捕获系统，已成为`LinkedIn`数据处理管道不可或缺的一部分。`Databus`解决了可靠捕获，流动和处理主要数据更改的基本要求。`Databus`提供以下功能：

  - 源与消费者之间的隔离
  - 保证按顺序和至少一次交付具有高可用性
  - 从更改流中的任意时间点开始消耗，包括整个数据的完全引导功能。
  - 分区消费
  - 源一致性保存

  ![1634720435220](E:\other\网络\assets\1634720435220.png)

  # **阿里云的数据传输服务DTS**

  数据传输服务（`Data Transmission Service`，简称DTS）是阿里云提供的一种支持 `RDBMS`（关系型数据库）、`NoSQL`、`OLAP `等多种数据源之间数据交互的数据流服务。DTS提供了数据迁移、实时数据订阅及数据实时同步等多种数据传输能力，可实现不停服数据迁移、数据异地灾备、异地多活(单元化)、跨境数据同步、实时数据仓库、查询报表分流、缓存更新、异步消息通知等多种业务应用场景，助您构建高安全、可扩展、高可用的数据架构。

  优势：数据传输（`Data Transmission`）服务 DTS 支持 RDBMS、NoSQL、OLAP 等多种数据源间的数据传输。它提供了数据迁移、实时数据订阅及数据实时同步等多种数据传输方式。相对于第三方数据流工具，数据传输服务 DTS 提供更丰富多样、高性能、高安全可靠的传输链路，同时它提供了诸多便利功能，极大得方便了传输链路的创建及管理。

  个人理解：就是一个消息队列，会给你推送它包装过的sql对象，可以自己做个服务去解析这些sql对象。

  免去部署维护的昂贵使用成本。`DTS`针对阿里云`RDS`（在线关系型数据库）、`DRDS`等产品进行了适配，解决了`Binlog`日志回收，主备切换、`VPC`网络切换等场景下的订阅高可用问题。同时，针对`RDS`进行了针对性的性能优化。出于稳定性、性能及成本的考虑，推荐使用。