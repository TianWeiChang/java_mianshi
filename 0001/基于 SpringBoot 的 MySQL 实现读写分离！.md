## 基于 SpringBoot 的 MySQL 实现读写分离！

大大

## 前言

首先思考一个问题:在高并发的场景中，关于数据库都有哪些优化的手段？常用的有以下的实现方法:读写分离、加缓存、主从架构集群、分库分表等，在互联网应用中，大部分都是**读多写少**的场景，设置两个库，主库和读库。

**主库的职能是负责写，从库主要是负责读，可以建立读库集群，通过读写职能在数据源上的隔离达到减少读写冲突、** 释压**数据库负载**、保护数据库**的目的**。在实际的使用中，凡是涉及到写的部分直接切换到主库，读的部分直接切换到读库，这就是典型的读写分离技术。本篇博文将聚焦读写分离，探讨如何实现它。

## 目录

- 一: 主从数据源的配置
- 二: 数据源路由的配置
- 三：数据源上下文环境
- 四：切换注解和 Aop 配置
- 五：用法以及测试
- 六：总结

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbuekHpHWMv94OMgfmttficibNqHakfVThumm7K9vYiacBo7T4HMj2yklxOVjQX8mGp36LCttlrJMNXHZw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**主从同步的局限性**：这里分为主数据库和从数据库，主数据库和从数据库保持数据库结构的一致，主库负责写，当写入数据的时候，会自动同步数据到从数据库；从数据库负责读，当读请求来的时候，直接从读库读取数据，主数据库会自动进行数据复制到从数据库中。不过本篇博客不介绍这部分配置的知识，因为它更偏运维工作一点。

这里涉及到一个问题:主从复制的延迟问题，当写入到主数据库的过程中，突然来了一个读请求，而此时数据还没有完全同步，就会出现读请求的数据读不到或者读出的数据比原始值少的情况。具体的解决方法最简单的就是将读请求暂时指向主库，但是同时也失去了主从分离的部分意义。也就是说在严格意义上的数据一致性场景中，读写分离并非是完全适合的，注意更新的时效性是读写分离使用的缺点。

好了，这部分只是了解，接下来我们看下具体如何通过 java 代码来实现读写分离:

该项目需要引入如下依赖：springBoot、spring-aop、spring-jdbc、aspectjweaver 等

## 一: 主从数据源的配置

我们需要配置主从数据库，主从数据库的配置一般都是写在配置文件里面。通过@ConfigurationProperties 注解，可以将配置文件(一般命名为:application.Properties)里的属性映射到具体的类属性上，从而读取到写入的值注入到具体的代码配置中，按照习惯大于约定的原则，主库我们都是注为 master，从库注为 slave。

本项目采用了阿里的 druid 数据库连接池，使用 build 建造者模式创建 DataSource 对象，DataSource 就是代码层面抽象出来的数据源，接着需要配置 sessionFactory、sqlTemplate、事务管理器等

```
/**
 * 主从配置
 *
 * @author wyq
 */
@Configuration
@MapperScan(basePackages = "com.wyq.mysqlreadwriteseparate.mapper"， sqlSessionTemplateRef = "sqlTemplate")
public class DataSourceConfig {

    /**
     * 主库
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource master() {
        return DruidDataSourceBuilder.create().build();
    }

    /**
     * 从库
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaver() {
        return DruidDataSourceBuilder.create().build();
    }


    /**
     * 实例化数据源路由
     */
    @Bean
    public DataSourceRouter dynamicDB(@Qualifier("master") DataSource masterDataSource，
                                      @Autowired(required = false) @Qualifier("slaver") DataSource slaveDataSource) {
        DataSourceRouter dynamicDataSource = new DataSourceRouter();
        Map<Object， Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceEnum.MASTER.getDataSourceName()， masterDataSource);
        if (slaveDataSource != null) {
            targetDataSources.put(DataSourceEnum.SLAVE.getDataSourceName()， slaveDataSource);
        }
        dynamicDataSource.setTargetDataSources(targetDataSources);
        dynamicDataSource.setDefaultTargetDataSource(masterDataSource);
        return dynamicDataSource;
    }


    /**
     * 配置sessionFactory
     * @param dynamicDataSource
     * @return
     * @throws Exception
     */
    @Bean
    public SqlSessionFactory sessionFactory(@Qualifier("dynamicDB") DataSource dynamicDataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setMapperLocations(
                new PathMatchingResourcePatternResolver().getResources("classpath*:mapper/*Mapper.xml"));
        bean.setDataSource(dynamicDataSource);
        return bean.getObject();
    }


    /**
     * 创建sqlTemplate
     * @param sqlSessionFactory
     * @return
     */
    @Bean
    public SqlSessionTemplate sqlTemplate(@Qualifier("sessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }


    /**
     * 事务配置
     *
     * @param dynamicDataSource
     * @return
     */
    @Bean(name = "dataSourceTx")
    public DataSourceTransactionManager dataSourceTransactionManager(@Qualifier("dynamicDB") DataSource dynamicDataSource) {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
        dataSourceTransactionManager.setDataSource(dynamicDataSource);
        return dataSourceTransactionManager;
    }
}
```

## 二: 数据源路由的配置

路由在主从分离是非常重要的，基本是读写切换的核心。Spring 提供了 AbstractRoutingDataSource 根据用户定义的规则选择当前的数据源，作用就是在执行查询之前，设置使用的数据源，实现动态路由的数据源，在每次数据库查询操作前执行它的抽象方法 determineCurrentLookupKey() 决定使用哪个数据源。

为了能有一个全局的数据源管理器，此时我们需要引入 DataSourceContextHolder 这个数据库上下文管理器，可以理解为全局的变量，随时可取(见下面详细介绍)，它的主要作用就是保存当前的数据源;

```
public class DataSourceRouter extends AbstractRoutingDataSource {

    /**
     * 最终的determineCurrentLookupKey返回的是从DataSourceContextHolder中拿到的，因此在动态切换数据源的时候注解
     * 应该给DataSourceContextHolder设值
     *
     * @return
     */
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.get();

    }
}
```

## 三：数据源上下文环境

数据源上下文保存器，便于程序中可以随时取到当前的数据源，它主要利用 ThreadLocal 封装，因为 ThreadLocal 是线程隔离的，天然具有线程安全的优势。这里暴露了 set 和 get、clear 方法，set 方法用于赋值当前的数据源名，get 方法用于获取当前的数据源名称，clear 方法用于清除 ThreadLocal 中的内容，因为 ThreadLocal 的 key 是 weakReference 是有内存泄漏风险的，通过 remove 方法防止内存泄漏；

```
/**
 * 利用ThreadLocal封装的保存数据源上线的上下文context
 */
public class DataSourceContextHolder {

    private static final ThreadLocal<String> context = new ThreadLocal<>();

    /**
     * 赋值
     *
     * @param datasourceType
     */
    public static void set(String datasourceType) {
        context.set(datasourceType);
    }

    /**
     * 获取值
     * @return
     */
    public static String get() {
        return context.get();
    }

    public static void clear() {
        context.remove();
    }
}
```

## 四：切换注解和 Aop 配置

首先我们来定义一个@DataSourceSwitcher 注解，拥有两个属性 ① 当前的数据源 ② 是否清除当前的数据源，并且只能放在方法上，(不可以放在类上，也没必要放在类上，因为我们在进行数据源切换的时候肯定是方法操作)，该注解的主要作用就是进行数据源的切换，在 dao 层进行操作数据库的时候，可以在方法上注明表示的是当前使用哪个数据源;

@DataSourceSwitcher 注解的定义:

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface DataSourceSwitcher {
    /**
     * 默认数据源
     * @return
     */
    DataSourceEnum value() default DataSourceEnum.MASTER;
    /**
     * 清除
     * @return
     */
    boolean clear() default true;

}
```

DataSourceAop 配置

为了赋予@DataSourceSwitcher 注解能够切换数据源的能力，我们需要使用 AOP，然后使用@Aroud 注解找到方法上有@DataSourceSwitcher.class 的方法，然后取注解上配置的数据源的值，设置到 DataSourceContextHolder 中，就实现了将当前方法上配置的数据源注入到全局作用域当中;

```
@Slf4j
@Aspect
@Order(value = 1)
@Component
public class DataSourceContextAop {

    @Around("@annotation(com.wyq.mysqlreadwriteseparate.annotation.DataSourceSwitcher)")
    public Object setDynamicDataSource(ProceedingJoinPoint pjp) throws Throwable {
        boolean clear = false;
        try {
            Method method = this.getMethod(pjp);
            DataSourceSwitcher dataSourceSwitcher = method.getAnnotation(DataSourceSwitcher.class);
            clear = dataSourceSwitcher.clear();
            DataSourceContextHolder.set(dataSourceSwitcher.value().getDataSourceName());
            log.info("数据源切换至：{}"， dataSourceSwitcher.value().getDataSourceName());
            return pjp.proceed();
        } finally {
            if (clear) {
                DataSourceContextHolder.clear();
            }

        }
    }

    private Method getMethod(JoinPoint pjp) {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        return signature.getMethod();
    }

}
```

## 五：用法以及测试

在配置好了读写分离之后，就可以在代码中使用了，一般而言我们使用在 service 层或者 dao 层，在需要查询的方法上添加@DataSourceSwitcher(DataSourceEnum.SLAVE)，它表示该方法下所有的操作都走的是读库;在需要 update 或者 insert 的时候使用@DataSourceSwitcher(DataSourceEnum.MASTER)表示接下来将会走写库。

其实还有一种更为自动的写法，可以根据方法的前缀来配置 AOP 自动切换数据源，比如 update、insert、fresh 等前缀的方法名一律自动设置为写库，select、get、query 等前缀的方法名一律配置为读库，这是一种更为自动的配置写法。缺点就是方法名需要按照 aop 配置的严格来定义，否则就会失效

```
@Service
public class OrderService {

    @Resource
    private OrderMapper orderMapper;


    /**
     * 读操作
     *
     * @param orderId
     * @return
     */
    @DataSourceSwitcher(DataSourceEnum.SLAVE)
    public List<Order> getOrder(String orderId) {
        return orderMapper.listOrders(orderId);

    }

    /**
     * 写操作
     *
     * @param orderId
     * @return
     */
    @DataSourceSwitcher(DataSourceEnum.MASTER)
    public List<Order> insertOrder(Long orderId) {
        Order order = new Order();
        order.setOrderId(orderId);
        return orderMapper.saveOrder(order);
    }
}
```

## 六：总结

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbuekHpHWMv94OMgfmttficibNqY5HpP6ibiaKdqYu5hVYhFNF2liaWRVtKicxusybpK91G0OzK7ticcyDwPkg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面是基本流程简图，本篇博客介绍了如何实现数据库读写分离，注意读写分离的核心点就是数据路由，需要继承 `AbstractRoutingDataSource，`复写它的 `determineCurrentLookupKey` 方法，同时需要注意全局的上下文管理器 `DataSourceContextHolder，`它是保存数据源上下文的主要类，也是路由方法中寻找的数据源取值，相当于数据源的中转站.再结合` jdbc-Template` 的底层去创建和管理数据源、事务等，我们的数据库读写分离就完美实现了。