分布式事务，`Seata`如何落地实践

# 前言

`Seata`是阿里巴巴研发的一套开源分布式事务框架，提供了`AT`、`TCC`、`SAGA `和` XA `几种事务模式。

本文以物流后台服务为例，介绍`Seata`框架落地的过程，遇到的问题以及解决方案。

我们从调研中发现，`Seata`框架既可以满足业务需求，灵活兼容多种事务模式，又可以实现数据强一致性。

# 1.  基础信息

- seata版本：1.4
- 微服务框架：springcloud
- 注册中心：consul

# 2.基本框架

## 2.1 基本组件

seata框架分为3个组件：

- **TC (Transaction Coordinator)** -事务协调者 （即seata-server）

维护全局和分支事务的状态，驱动全局事务提交或回滚。

- **TM (Transaction Manager)** -事务管理器 （在client上，发起事务的服务）

定义全局事务的范围：开始全局事务、提交或回滚全局事务。

- **RM (Resource Manager) -** 资源管理器 （在client）

管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚

## 2.2. 部署seata-server(TC)

在官网下载 seata 服务端，解压后执行bin/seata-server.sh即可启动。

seata-server 有2个配置文件：registry.conf 与 file.conf。而 registry.conf 文件决定了 seata-server 使用的注册中心配置和配置信息获取方式。

我们使用 consul 做注册中心，因此需要在registry.conf文件中，需要修改以下配置：

```groovy
registry {
  #file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "consul" ## 这里注册中心填consul
  loadBalance = "RandomLoadBalance"
  loadBalanceVirtualNodes = 10
   ... ...
  consul {
    cluster = "seata-server"
    serverAddr = "***注册中心地址***"
    #这里的dc指的是datacenter，若consul为多数据源配置需要在请求中加入dc参数。
    #dc与namespace并非是seata框架自带的，文章后面将会进一步解释
    dc="bj-th"
    namespace="seata-courseop"
  }
  ... ...
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  ## 如果启动时从注册中心获取基础配置信息，填consul
  ## 否则从file.conf文件中获取
  type = "consul"
  consul {
    serverAddr = "127.0.0.1:8500"
  }
... ...
}
```

其中需要注意的是，如果需要高可用部署，seata获取配置信息的方式就必须是注册中心，此时file.conf就没用了。

（当然，需要事先把file.conf文件中的配置信息迁移到consul中）

```groovy
store {
  ## store mode: file、db、redis
  mode = "db"

... ...
  ## database store property
  ## 如果使用数据库模式，需要配置数据库连接设置
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://***线上数据库地址***/seata"
    user = "******"
    password = "******"
    minConn = 5
    maxConn = 100
    ## 这里的三张表需要提前在数据库建好
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
... ...
}

service {
  #vgroup->rgroup
  vgroupMapping.tx-seata="seata-server"
  default.grouplist="127.0.0.1:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
```

其中，**global_table**， **branch_table**，**lock_table**三张表需要提前在数据库中建好。

## 2.3 配置client端（RM与TM）

每个使用seata框架的服务都需要引入seata组件：

```groovy
dependencies {

    api 'com.alibaba:druid-spring-boot-starter:1.1.10'
    api 'mysql:mysql-connector-java:6.0.6'
    api('com.alibaba.cloud:spring-cloud-alibaba-seata:2.1.0.RELEASE') {
        exclude group:'io.seata', module:'seata-all'
    }
    api 'com.ecwid.consul:consul-api:1.4.5'
    api 'io.seata:seata-all:1.4.0'
}

dependencies {

    api 'com.alibaba:druid-spring-boot-starter:1.1.10'
    api 'mysql:mysql-connector-java:6.0.6'
    api('com.alibaba.cloud:spring-cloud-alibaba-seata:2.1.0.RELEASE') {
        exclude group:'io.seata', module:'seata-all'
    }
    api 'com.ecwid.consul:consul-api:1.4.5'
    api 'io.seata:seata-all:1.4.0'
}
```

每个服务都同样需要配置file.conf与registry.conf文件，放在resource目录下。registry.conf与server的保持一致。在file.conf文件中，除了db配置外，还需要进行client参数的配置：

```groovy
client {
  rm {
    asyncCommitBufferLimit = 10000
    lock {
      retryInterval = 10
      retryTimes = 30
      retryPolicyBranchRollbackOnConflict = true
    }
    reportRetryCount = 5
    tableMetaCheckEnable = false
    reportSuccessEnable = false
  }
  tm {
    commitRetryCount = 5
    rollbackRetryCount = 5
  }
  undo {
    dataValidation = true
    logSerialization = "jackson"
    ## 这个undo_log也需要提前在mysql中创建
    logTable = "undo_log"
  }
  log {
    exceptionRate = 100
  }
}
```

在application.yml文件中添加seata配置：

```yaml
spring:
  cloud:
      seata: ## 注意tx-seata需要与服务端和客户端的配置文件保持一致
        tx-service-group: tx-seata
```

另外，还需要替换项目的数据源，

```java
@Primary
@Bean("dataSource")
public DataSource druidDataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setUrl(url);
        druidDataSource.setUsername(username);
        druidDataSource.setPassword(password);
        druidDataSource.setDriverClassName(driverClassName);至此，client端的配置也已经完成了。
        return new DataSourceProxy(druidDataSource);
}
```

至此，client端的配置也已经完成了。

# 3. 功能演示

一个分布式的全局事务，整体是两阶段提交的模型。

**全局事务**是由若干分支事务组成的，

**分支事务**要满足两阶段提交的模型要求，即需要每个分支事务都具备自己的：

- 一阶段 prepare 行为
- 二阶段commit 或 rollback 行为

根据两阶段行为模式的不同，我们将分支事务划分为 **Automatic (Branch) Transaction Mode 和 TCC (Branch) Transaction Mode.**

## 3.1 AT模式

AT 模式基于支持本地ACID事务的关系型数据库：

- 一阶段**prepare**行为：在本地事务中，一并提交业务数据更新和相应回滚日志记录。
- 二阶段 **commit** 行为：马上成功结束，自动异步批量清理回滚日志。
- 二阶段 **rollback** 行为：通过回滚日志，自动生成补偿操作，完成数据回滚

> 直接在需要添加全局事务的方法中加上注解@GlobalTransactional

```java
@SneakyThrows
@GlobalTransactional
@Transactional(rollbackFor = Exception.class)
public void buy(int id, int itemId){
        // 先生成订单
        Order order = orderFeignDao.create(id, itemId);
        // 根据订单扣减账户余额
        accountFeignDao.draw(id, order.amount);
}
```

> 注意：同@Transactional一样，@GlobalTransactional若要生效也要满足：

- 目标函数必须为public类型
- 同一类内方法调用时，调用目标函数的方法必须通过springBeanName.method的形式来调用，不能使用this直接调用内部方法

## 3.2TCC模式

TCC 模式是支持把自定义的分支事务纳入到全局事务的管理中。

- 一阶段 prepare 行为：调用自定义的 prepare 逻辑。
- 二阶段 commit 行为：调用自定义的 commit 逻辑。
- 二阶段 rollback 行为：调用自定义的 rollback 逻辑。

> **首先**编写一个TCC服务接口:

```
@LocalTCC
public interface BusinessAction {
    @TwoPhaseBusinessAction(name = "doBusiness", commitMethod = "commit", rollbackMethod = "rollback")
    boolean doBusiness(BusinessActionContext businessActionContext,
                       @BusinessActionContextParameter(paramName = "message") String msg);

    boolean commit(BusinessActionContext businessActionContext);

    boolean rollback(BusinessActionContext businessActionContext);
}
```

> 其中，BusinessActionContext为全局事务上下文，可以从此对象中获取全局事务相关信息（如果是发起全局事务方，传入null后自动生成），然后实现该接口:

```java
@Slf4j
@Service
public class BusinessActionImpl implements BusinessAction {

    @Transactional(rollbackFor = Exception.class)
    @Override
    public boolean doBusiness(BusinessActionContext businessActionContext, String msg) {
        log.info("准备do business:{}",msg);
        return true;
    }

    @Transactional(rollbackFor = Exception.class)
    @Override
    public boolean commit(BusinessActionContext businessActionContext) {
        log.info("business已经commit");
        return true;
    }

    @Transactional(rollbackFor = Exception.class)
    @Override
    public boolean rollback(BusinessActionContext businessActionContext) {
        log.info("business已经rollback");
        return true;
    }
}
```

> **最后**，开启全局事务方法同AT模式。

```java
@SneakyThrows
@GlobalTransactional
public void doBusiness(BusinessActionContext context, String msg){
        accountFeignDao.draw(3, new BigDecimal(100));
        businessAction.doBusiness(context, msg);
}
```

# 4. 遇到的问题

## 4.1 client TM/RM 无法注册到TC

在部署seata项目时常常会遇到这样的问题：在本地调试时一切正常，但是当试图部署到线上时，总是在clinet端提示注册TC端失败。

- 这是因为client需要先通过服务发现，找到注册中心里seata-server的服务信息，然后再与seata-server建立连接。不过线上的consul采用了多数据中心模式，在调用consul api时，必须加上dc参数项，否则将无法返回正确的服务信息；然而，seata提供的consul服务发现组件似乎并不支持dc参数的配置。
- 还有一个原因也会导致client无法连接到TC：seata的consul客户端在调用服务状态监控api时，使用了wait与index参数，从而使consul查询进入了阻塞查询模式。此时client对consul中要查询的key做监听，只有当key发生变化或者达到最大请求时间时，才会返回结果。貌似由于consul版本的问题，这个阻塞查询并没有监听到key的变化，反而会让服务发现的线程陷入无限等待之中，自然也就无法让client获取到server的注册信息了。

## 4.2高可用部署

`Seata`服务的高可用部署**只支持注册中心模式**。因此，我们需要想办法将`file.conf`文件以键值对的形式存到consul中。

遗憾的是，`consul`并没有显式支持`namespace`，我们只能在put请求中用“/”为分隔符起到类似的效果。当然，seata框架也没有考虑到这一点。所以我们需要修改源码中的`Configuration`接口与`RegistryProvider`接口的consul实现类，增加`namespace`属性。

## 4.3 global_log与branch_log

TC在想mysql插入日志数据时，偶尔会报：

```
Caused by: java.sql.SQLException: Incorrect string value:
```

application_data字段其实就是对业务数据的记录。官方给出的建表语句是这样的：

```sql
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

显然，VARCHAR(2000)的大小是不合适的， utf8的格式也是不合适的。所以我们需要修改seata关于**数据源连接**的部分代码：

```java
// connectionInitSql设置
protected Set<String> getConnectionInitSqls(){
        Set<String> set = new HashSet<>();
        String connectionInitSqls = CONFIG.getConfig(ConfigurationKeys.STORE_DB_CONNECTION_INIT_SQLS);
        if(StringUtils.isNotEmpty(connectionInitSqls)) {
            String[] strs = connectionInitSqls.split(",");
            for(String s:strs){
                set.add(s);
            }
        }
        // 默认支持utf8mb4
        set.add("set names utf8mb4");
        return set;
}
```

# 5.自定义开发

## 5.1利用SPI机制编写自定义组件

`Seata`基于java的spi机制提供了自定义实现接口的功能，我们只需要在自己的服务中，根据seata的接口写好自己的实现类即可。

> `SPI（Service Provider Interface）`是JDK内置的服务发现机制，用在不同模块间通过接口调用服务，避免对具体服务服务接口具体实现类的耦合。比如JDBC的数据库驱动模块，不同数据库连接驱动接口相同但实现类不同，在使用SPI机制以前调用驱动代码需要直接在类里采用`Class.forName(具体实现类全名）`的方式调用，这样调用方依赖了具体的驱动实现，在替换驱动实现时要修改代码。

以**`ConsulRegistryProvider`**为例：

- `ConsulRegistryServiceImpl`

```java
// 增加DC和namespace
private static String NAMESPACE;
private static String DC;

private ConsulConfiguration() {
        Config registryCongig = ConfigFactory.parseResources("registry.conf");
        NAMESPACE = registryCongig.getString("config.consul.namespace");
        DC = CommonSeataConfiguration.getDatacenter();
        consulNotifierExecutor = new ThreadPoolExecutor(THREAD_POOL_NUM, THREAD_POOL_NUM, Integer.MAX_VALUE,
                TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(),
                new NamedThreadFactory("consul-config-executor", THREAD_POOL_NUM));
}
    ... ...
// 同时在getHealthyServices中，删除请求参数wait&index    
/**
  * get healthy services
  *
  * @param service
  * @return
*/
private Response<List<HealthService>> getHealthyServices(String service, long index, long watchTimeout) {
        return getConsulClient().getHealthServices(service, HealthServicesRequest.newBuilder()
                .setTag(SERVICE_TAG)
                .setDatacenter(DC)
                .setPassing(true)
                .build());
}
```

- ConsulRegistryProvider 注意order要大于seata包中的默认值1，seata类加载器会优先加载order更大的实现类

```java
@LoadLevel(name = "Consul" ,order = 2)
public class ConsulRegistryProvider implements RegistryProvider {
    @Override
    public RegistryService provide() {
        return ConsulRegistryServiceImpl.getInstance();
    }
}
```

- 然后在META-INF 的services目录下添加`io.seata.discovery.registry.RegistryProvider`

```
com.youdao.ke.courseop.common.seata.ConsulRegistryProvider
```

这样就可以替换seata包中的实现了。

## 5.2 common-seata工具包

对于这些自定义实现类，以及一些公共client配置，我们可以统一封装到一个工具包下：

![图1.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab52a796edde4b819bb31a5f715a3166~tplv-k3u1fbpfcp-watermark.awebp)

这样，其他项目只需要引入这个工具包，就可以无需繁琐的配置，直接使用了。

> gradle引入common包：

```
api 'com.youdao.ke.courseop.common:common-seata:0.0.+'
```

# 6. 落地实例

以一个物流场景为例： **业务架构**：

- logistics-server （物流服务）
- logistics-k3c-server (物流-金蝶客户端，封装调用金蝶服务的api）
- elasticsearch

**业务背景**：logistics 执行领用单新增，在 elasticsearch 中更新数据，同时通过 rpc 调用 logistics-k3c 的金蝶出库方法，生成金蝶单据，

![图2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/20da70c7bb8c48ee8898e50a8a31eea4~tplv-k3u1fbpfcp-watermark.awebp)

**问题**：如果elasticsearch单据更新出现异常，金蝶单据将无法回滚，造成数据不一致的问题。

在部署完seata线上服务后，只需要在logistics与logistics-k3c中分别引入**common-seata工具包**

**logistics服务**：

```java
// 使用全局事务注解开启全局事务
@GlobalTransactional
@Transactional(rollbackFor = Exception.class)
public void Scm通过(StaffOutStockDoc staffOutStock, String body) throws Exception {
        ... 一些业务处理...
         // 构建金蝶单据请求
        K3cApi.StaffoutstockReq req = new K3cApi.StaffoutstockReq();
        req.materialNums = materialNums;
        req.staffOutStockId = staffOutStock.id;
        ... 一些业务处理 ...
       // 调用logistics-k3c-api金蝶出库
        k3cApi.staffoutstockAuditPass(req);

        staffOutStock.status = 待发货;
        staffOutStock.scmAuditTime = new Date();
        staffOutStock.updateTime = new Date();
        staffOutStock.historyPush("scm通过");
        // 更新对象后存入elasticsearch
        es.set(staffOutStock);
}
```

**logistics-k3c**：

由于我们新增单据接口是调用金蝶的服务，所以这里使用TCC模式构建事务接口

- 首先创建StaffoutstockCreateAction接口

```java
@LocalTCC
public interface StaffoutstockCreateAction {
    @TwoPhaseBusinessAction(name = "staffoutstockCreate")
    boolean create(BusinessActionContext businessActionContext,
                       @BusinessActionContextParameter(paramName = "staffOutStock") StaffOutStock staffOutStock,
                       @BusinessActionContextParameter(paramName = "materialNum") List<Triple<Integer, Integer, Integer>> materialNum);

    boolean commit(BusinessActionContext businessActionContext);

    boolean rollback(BusinessActionContext businessActionContext);

}
```

- 接口实现StaffoutstockCreateActionImpl

```java
@Slf4j
@Service
public class StaffoutstockCreateActionImpl implements StaffoutstockCreateAction {

    @Autowired
    private K3cAction4Staffoutstock k3cAction4Staffoutstock;

    @SneakyThrows
    @Transactional(rollbackFor = Exception.class)
    @Override
    public boolean create(BusinessActionContext businessActionContext, StaffOutStock staffOutStock, List<Triple<Integer, Integer, Integer>> materialNum) {
        //金蝶单据新增
        k3cAction4Staffoutstock.staffoutstockAuditPass(staffOutStock, materialNum);
        return true;
    }

    @SneakyThrows
    @Transactional(rollbackFor = Exception.class)
    @Override
    public boolean commit(BusinessActionContext businessActionContext) {
        Map<String, Object> context = businessActionContext.getActionContext();
        JSONObject staffOutStockJson = (JSONObject) context.get("staffOutStock");
        // 如果尝试新增成功，commit不做任何事
        StaffOutStock staffOutStock = staffOutStockJson.toJavaObject(StaffOutStock.class);
        log.info("staffoutstock {} commit successfully!", staffOutStock.id);
        return true;
    }

    @SneakyThrows
    @Transactional(rollbackFor = Exception.class)
    @Override
    public boolean rollback(BusinessActionContext businessActionContext) {
        Map<String, Object> context = businessActionContext.getActionContext();
        JSONObject staffOutStockJson = (JSONObject) context.get("staffOutStock");
        StaffOutStock staffOutStock = staffOutStockJson.toJavaObject(StaffOutStock.class);
        // 这里调用金蝶单据删除接口进行回滚
        k3cAction4Staffoutstock.staffoutstockRollback(staffOutStock);
        log.info("staffoutstock {} rollback successfully!", staffOutStock.id);
        return true;
    }
}
```

- 封装为业务方法

```java
/**
 * 项目组领用&报废的审核通过：新增其他出库单
 * 该方法使用seata-TCC方案实现全局事务
 * @param staffOutStock
 * @param materialNum
 */
    
@Transactional
public void staffoutstockAuditPassWithTranscation(StaffOutStock staffOutStock,
                                                      List<Triple<Integer, Integer, Integer>> materialNum){
        staffoutstockCreateAction.create(null, staffOutStock, materialNum);
}
```

- k3c API实现类

```java
 @SneakyThrows
 @Override
 public void staffoutstockAuditPass(StaffoutstockReq req) {
        ... 一些业务处理方法 ...
        //这里调用了封装好的事务方法
        k3cAction4Staffoutstock.staffoutstockAuditPassWithTranscation(staffOutStock, triples);
 }
```

这样，一个**基于TCC的全局事务链路**就建立起来了。

当全局事务**执行成功**时，我们可以在 server 中看到打印的日志：

![图3.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a40593cc2b14e72b67da62ebe2569b1~tplv-k3u1fbpfcp-watermark.awebp)

如果全局事务**执行失败**，会进行回滚，此时会执行接口中的rollback，调用金蝶接口删除生成的单据，

![图4.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/857016c286a949e188b6704983b3ba62~tplv-k3u1fbpfcp-watermark.awebp)

# 7. 总结

本文以seata框架的部署与使用为主线，记录了**seata**框架运用的一些**关键步骤与技术细节**，并针对项目落地时遇到的一些的技术问题提供了解决方案。

 

 

 

 