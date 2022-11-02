项目中疯狂使用SPI思想，在这里总结下 

前段时间项目中疯狂使用SPI思想，在这里总结下。

## 为什么要使用spi

面向的对象的设计里，我们一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。为了实现在模块装配的时候不用在程序里动态指明，这就需要一种服务发现机制。java spi就是提供这样的一个机制：为某个接口寻找服务实现的机制。这有点类似IOC的思想，将装配的控制权移到了程序之外。

以上文字从别处复制而来，想必你还是一脸懵逼，但不要慌，去搜一下spi你就会感觉更懵逼，因为你搜出来的只会是这个：

![img](https://pic2.zhimg.com/80/v2-df75c54974350155e5680829fa2e6a91_720w.png)

那到底啥是spi思想呢？

## spi的概念

![img](https://pic2.zhimg.com/80/v2-e755e1d56a1bc819f7a1b02a6bda5d81_720w.png)

首先放个图：我们在“调用方”和“实现方”之间需要引入“接口”，可以思考一下什么情况应该把接口放入调用方，什么时候可以把接口归为实现方。

先来看看接口属于实现方的情况，这个很容易理解，实现方提供了接口和实现，我们可以引用接口来达到调用某实现类的功能，这就是我们经常说的api，它具有以下特征：

1. 概念上更接近实现方
2. 组织上位于实现方所在的包中
3. 实现和接口在一个包中

当接口属于调用方时，我们就将其称为spi，全称为：service provider interface，spi的规则如下：

1. 概念上更依赖调用方
2. 组织上位于调用方所在的包中
3. 实现位于独立的包中（也可认为在提供方中）

如下图所示：

![img](https://pic1.zhimg.com/80/v2-5210ed7234f7046273104bed9bfd7764_720w.png)

## 开源的案例

接下来从几个案例总结下java spi思想

## Jdk

在jdk6里面引进的一个新的特性ServiceLoader，从官方的文档来说，它主要是用来装载一系列的service provider。而且ServiceLoader可以通过service provider的配置文件来装载指定的service provider。当服务的提供者，提供了服务接口的一种实现之后，我们只需要在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。
可能上面讲的有些抽象，下面就结合一个示例来具体讲讲。

## jdk spi案例

我们现在需要使用一个内容搜索接口，搜索的实现可能是基于文件系统的搜索，也可能是基于数据库的搜索。

先定义好接口

```
package com.cainiao.ys.spi.learn;

import java.util.List;

public interface Search {
    public List<String> searchDoc(String keyword);   
}
```

文件搜索实现

```
package com.cainiao.ys.spi.learn;

import java.util.List;

public class FileSearch implements Search{
    @Override
    public List<String> searchDoc(String keyword) {
        System.out.println("文件搜索 "+keyword);
        return null;
    }
}
```

数据库搜索实现

```
package com.cainiao.ys.spi.learn;

import java.util.List;

public class DatabaseSearch implements Search{
    @Override
    public List<String> searchDoc(String keyword) {
        System.out.println("数据搜索 "+keyword);
        return null;
    }
}
```

接下来可以在resources下新建META-INF/services/目录，然后新建接口全限定名的文件：com.cainiao.ys.spi.learn.Search，里面加上我们需要用到的实现类

```
com.cainiao.ys.spi.learn.FileSearch
```

然后写一个测试方法

```
package com.cainiao.ys.spi.learn;

import java.util.Iterator;
import java.util.ServiceLoader;

public class TestCase {
    public static void main(String[] args) {
        ServiceLoader<Search> s = ServiceLoader.load(Search.class);
        Iterator<Search> iterator = s.iterator();
        while (iterator.hasNext()) {
           Search search =  iterator.next();
           search.searchDoc("hello world");
        }
    }
}
```

可以看到输出结果：文件搜索 hello world

如果在com.cainiao.ys.spi.learn.Search文件里写上两个实现类，那最后的输出结果就是两行了。
这就是因为ServiceLoader.load(Search.class)在加载某接口时，会去META-INF/services下找接口的全限定名文件，再根据里面的内容加载相应的实现类。

这就是spi的思想，接口的实现由provider实现，provider只用在提交的jar包里的META-INF/services下根据平台定义的接口新建文件，并添加进相应的实现类内容就好。

那为什么配置文件为什么要放在META-INF/services下面？
可以打开ServiceLoader的代码，里面定义了文件的PREFIX如下：

```
private static final String PREFIX = "META-INF/services/"
```

以上是我们自己的实现，接下来可以看下jdk中DriverManager的spi设计思路

## DriverManager spi案例

DriverManager是jdbc里管理和注册不同数据库driver的工具类。从它设计的初衷来看，和我们前面讨论的场景有相似之处。针对一个数据库，可能会存在着不同的数据库驱动实现。我们在使用特定的驱动实现时，不希望修改现有的代码，而希望通过一个简单的配置就可以达到效果。

我们在使用mysql驱动的时候，会有一个疑问，DriverManager是怎么获得某确定驱动类的？

我们在运用Class.forName("com.mysql.jdbc.Driver")加载mysql驱动后，就会执行其中的静态代码把driver注册到DriverManager中，以便后续的使用。
代码如下：

```
package com.mysql.jdbc;

public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

这里可以看到，不同的驱动实现了相同的接口java.sql.Driver，然后通过registerDriver把当前driver加载到DriverManager中
这就体现了使用方提供规则，提供方根据规则把自己加载到使用方中的spi思想

这里有一个有趣的地方，查看DriverManager的源码，可以看到其内部的静态代码块中有一个loadInitialDrivers方法，在注释中我们看到用到了上文提到的spi工具类ServiceLoader

```
/**
* Load the initial JDBC drivers by checking the System property
* jdbc.properties and then use the {@code ServiceLoader} mechanism
*/
static {
	loadInitialDrivers();
	println("JDBC DriverManager initialized");
}
```

点进方法，看到方法里有如下代码：

```
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
Iterator<Driver> drivers = loadedDrivers.iterator();
println("DriverManager.initialize: jdbc.drivers = " + loadedDrivers);
```

可见，DriverManager初始化时也运用了spi的思想，使用ServiceLoader把写到配置文件里的Driver都加载了进来。

我们打开mysql-connector-java的jar包，果然在META-INF/services下发现了上文中提到的接口路径，打开里面的内容，可以看到是com.mysql.jdbc.Driver

![img](https://pic1.zhimg.com/80/v2-205e1d0141a7120b71e3c579340ed260_720w.png)

其实对符合DriverManager设定规则的驱动，我们并不用去调用class.forname，直接连接就好.因为DriverManager在初始化的时候已经把所有符合的驱动都加载进去了，避免了在程序中频繁加载。

但对于没有符合配置文件规则的驱动，如oracle，它还是需要去显示调用classforname，再执行静态代码块把驱动加载到manager里，因为它不符合配置文件规则:

![img](https://pic4.zhimg.com/80/v2-4031db98251f12be0943217beeaa5d3b_720w.png)

最后总结一下jdk spi需要遵循的规范

![img](https://pic3.zhimg.com/80/v2-9db1c292ae743dad89615c2a41d967b6_720w.png)

## 插件体系

## eclipse插件

其实最具spi思想的应该属于插件开发，我们项目中也用到的这种思想，后面再说，这里具体说一下eclipse的插件思想。

Eclipse使用OSGi作为插件系统的基础，动态添加新插件和停止现有插件，以动态的方式管理组件生命周期。

一般来说，插件的文件结构必须在指定目录下包含以下三个文件：

- META-INF/MANIFEST.MF: 项目基本配置信息，版本、名称、启动器等
- build.properties: 项目的编译配置信息，包括，源代码路径、输出路径
- plugin.xml：插件的操作配置信息，包含弹出菜单及点击菜单后对应的操作执行类等

当eclipse启动时，会遍历plugins文件夹中的目录，扫描每个插件的清单文件MANIFEST.MF，并建立一个内部模型来记录它所找到的每个插件的信息，就实现了动态添加新的插件。

这也意味着是eclipse制定了一系列的规则，像是文件结构、类型、参数等。插件开发者遵循这些规则去开发自己的插件，eclipse并不需要知道插件具体是怎样开发的，只需要在启动的时候根据配置文件解析、加载到系统里就好了，是spi思想的一种体现。

## Spring

Spring中运用到spi思想的地方也有很多，下面随便列举几个

## scan

我们在spring中可以通过component-scan标签来对指定包路径进行扫描，只要扫到spring制定的@service、@controller等注解，spring自动会把它注入容器。

这就相当于spring制定了注解规范，我们按照这个注解规范开发相应的实现类或controller，spring并不需要感知我们是怎么实现的，他只需要根据注解规范和scan标签注入相应的bean，这正是spi理念的体现。

## Scope

spring中有作用域scope的概念。
除了singleton、prototype、request、session等spring为我们提供的域，我们还可以自定义scope。

像是自定义一个 ThreadScope实现Scope接口
再把它注册到beanFactory中

```
Scope threadScope = new ThreadScope();
beanFactory.registerScope("thread", threadScope);
```

接着就能在xml中使用了

```
<bean id=".." class=".." scope="thread"/>
```

spring作为使用方提供了自定义scope的规则，提供方根据规则进行编码和配置，这样在spring中就能运用我们自定义的scope了，并不需要spring感知我们scope里的实现，这也是平台使用方制定规则，提供方负责实现的思想。

## 自定义标签

扩展Spring自定义标签配置大致需要以下几个步骤

1. 创建一个需要扩展的组件，也就是一个bean
2. 定义一个XSD文件描述组件内容，也可以给bean的属性赋值啥的
3. 创建一个文件，实现BeanDefinitionParser接口，用来解析XSD文件中的定义和对组件进行初始化，像是为组件bean赋上xsd里设置的值
4. 创建一个Handler文件，扩展自NamespaceHandlerSupport，目的是将组件注册到Spring容器，重写其中的的init方法

这样我们就边写出了一个自定义的标签，spring只是为我们定义好了创建标签的流程，不用感知我们是如何实现的，我们通过register就把自定义标签加载到了spring中，实现了spi的思想。

## ConfigurableBeanFactory

spring里为我们提供了许多属性编辑器，这时我们如果想把spring配置文件中的字符串转换成相应的对象进行注入，就要自定义属性编辑器，这时我们可以按照spring为我们提供的规则来自定义我们的编辑器

自定义好了属性编辑器后，ConfigurableBeanFactory里面有一个registerCustomEditor方法，此方法的作用就是注册自定义的编辑器，也是spi思想的体现

## Hotspot

我们打开hotspot的源码，可以看到，代码分为shared和其他，shared属于引擎层，os属于不同行业的实现

![img](https://pic4.zhimg.com/80/v2-da1f30b00934c824911b43147fbeb42f_720w.png)

点开os，可以发现里面有不同操作系统的不同实现：

![img](https://pic3.zhimg.com/80/v2-dd20fb6fd411ec4a2c3e642b128e725e_720w.png)

不同的厂商会提供hotspot的不同实现，在hotspot启动的时候，会判断当前是什么系统来启动不同的实现，这也是一种spi的思想。

在hotspot启动时会去执行shared里的代码，除了shared的其他三个包相当于是外部的一些实现，是不同操作系统开发人员加载到hotspot中，这种分层思想已经算是spi的思想
接着我们可以在shared里看到一个createThread接口，这个接口在不同的os下实现肯定是不一样的，这就代表着hotspot制定接口，不同的os开发者去捐献实现，hotspot不用感知是如何实现的，只需要在运行时直接调用接口就好，也是spi的思想。

还有像是`Jetty/Tomcat`中自定义`sessionManager`、自定义线程池，Dubbo的扩展机制等内容也属于spi。

## 总结

其实在这里就可以发现，只要是能满足用户按照系统规则来自定义，并且可以注册到系统中的功能点，都带有着spi的思想