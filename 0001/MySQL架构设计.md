MySQL架构设计

对于大部分的开发人员而言，编写增删查改的sql语句通过数据库连接去操作数据库，但并不关心数据库是如何监听请求和从连接中把请求数据中提取出来，往往在意表结构，sql执行效率慢就给他们建立索引，完全把MySQL当作黑盒子去使用。

### 1. 网络连接必须使用线程来处理

MySQL 使用内部线程来实现监听和读取请求。

![图片](https://mmbiz.qpic.cn/mmbiz_png/OKUeiaP72uRwXfbHoJSG7daickdfsQRIzgyTBkpOxMlQTnoHicILYd3kFjiaXq19kCNGMD9HAXRqKMfeiaGicoiaO1hgg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2. SQL接口：负责处理接收到的sql语句

MySQL通过sql接口把我们平时编写的sql语句简单化，让我们轻松的学会和编写sql语句，但其底层实现其实是非常复杂。当工作线程接收到sql语句之后，会交给sql接口去执行。

![图片](https://mmbiz.qpic.cn/mmbiz_png/OKUeiaP72uRwXfbHoJSG7daickdfsQRIzgaL4KdzwUJ1VPicWLrlIdkjFWN4YPUHuVYfRP3AUWVpicceEYd8q2Y7bg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3. 查询解释器：让MySQL看懂sql语句

MySQL是一个数据管理系统，并不能像我们一样直接读懂sql语句，例如:

```
select id, name, age from users where id = 1
```

需要借助组件 查询解析器 对sql语句进行解析和拆解，拆解成以下几部分：

1. from users: 我们需要从 users 表里面查询数据
2. where id = 1 ：查询id字段值为1的那行数据
3. select id, name, age : 从查出来的那行数据中提取出 "id,name,age"三个字段

![图片](https://mmbiz.qpic.cn/mmbiz_png/OKUeiaP72uRwXfbHoJSG7daickdfsQRIzg0vAThh4lCibCCYMVqkVAn74rdmuPTYlbj3wgV4aBiajMW5VOBG2UpaUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4. 查询优化器：选择最优的查询路径

查询优化器会根据sql生成查询路径树，然后从里面选择一条最优的查询路径出来。

![图片](https://mmbiz.qpic.cn/mmbiz_png/OKUeiaP72uRwXfbHoJSG7daickdfsQRIzgQsXvL13c4LCibAxZSxMMCPhia2E0PVKEcbF8Kmia0tBqW2G7XSc9bCt4Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 5. 调用存储引擎接口，真正执行sql语句

数据库存储的数据，有可能存储在磁盘上，有可能存储在内存中。那怎么判别查询的数据存放在哪一个地方？存储引擎根据执行器的调度执行sql逻辑，无论是从内存缓存中查询数据，从磁盘中更新数据，一系列操作全都由存储引擎执行。

![图片](https://mmbiz.qpic.cn/mmbiz_png/OKUeiaP72uRwXfbHoJSG7daickdfsQRIzgblZImplia10EJaNbYcia7XzXHD3WpL0iaCbicRnicKndJxpicZwxvNtZ3Bag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 6. 执行器：根据执行计划调度存储引擎

执行器根据优化器的执行计划去调用存储引擎的各种接口来完成sql语句的执行。

![图片](https://mmbiz.qpic.cn/mmbiz_png/OKUeiaP72uRwXfbHoJSG7daickdfsQRIzgBFpFTjnLibzwsaqaecMVEyjP88KDyFksJmicTpGp2kWLY05ibOZkqWf0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结

在MySQL架构设计中，SQL接口、SQL解析器、查询优化器、执行器都是一套通用的组件，但是存储引擎却有不同的选择，例如：InnoDB、MyISAM、Memory等，对应不同的应用场景，MySQL的默认是 InnoDB，在后续会一步一步分析。所以综上所述，MySQL的执行sql语句的顺序为：sql接口->解析器：解释sql->优化器：生成执行计划->执行器：执行计划去调用InnoDB存储引擎接口执行sql