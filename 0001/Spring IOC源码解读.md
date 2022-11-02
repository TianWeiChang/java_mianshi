Spring IOC源码解读



今天我们分享的是IOC源码解读，文章很长，希望静下心来，好好看，绝对收获大大的。

> 请耐心看完



## 主要内容

文章一共有五大部分，具体如下：

### 一、 什么是Ioc/DI？

### 二、 Spring IOC体系结构

(1) `BeanFactory`

(2) `BeanDefinition`

### 三、 IoC容器的初始化

1、` XmlBeanFactory`(屌丝IOC)的整个流程

2、` FileSystemXmlApplicationContext `的IOC容器流程

2.1、高富帅`IOC`解剖

2.2、 设置资源加载器和资源定位

2.3、AbstractApplicationContext的refresh函数载入Bean定义过程：

2.4、AbstractApplicationContext子类的refreshBeanFactory()方法：

2.5、AbstractRefreshableApplicationContext子类的loadBeanDefinitions方法：

2.6、AbstractBeanDefinitionReader读取Bean定义资源：

2.7、资源加载器获取要读入的资源：

2.8、XmlBeanDefinitionReader加载Bean定义资源：

2.9、DocumentLoader将Bean定义资源转换为Document对象：

2.10、XmlBeanDefinitionReader解析载入的Bean定义资源文件：

2.11、DefaultBeanDefinitionDocumentReader对Bean定义的Document对象解析：

2.12、BeanDefinitionParserDelegate解析Bean定义资源文件中的`<Bean>`元素：

2.13、BeanDefinitionParserDelegate解析`<property>`元素：

2.14、解析`<property>`元素的子元素：

2.15、解析`<list>`子元素：

2.16、解析过后的BeanDefinition在IoC容器中的注册：

2.17、DefaultListableBeanFactory向IoC容器注册解析后的BeanDefinition：

总结：

###  四、IOC容器的依赖注入

1、依赖注入发生的时间

2、AbstractBeanFactory通过getBean向IoC容器获取被管理的Bean：

3、AbstractAutowireCapableBeanFactory创建Bean实例对象：

4、createBeanInstance方法创建Bean的java实例对象：

5、SimpleInstantiationStrategy类使用默认的无参构造方法创建Bean实例化对象：

6、populateBean方法对Bean属性的依赖注入：

7、BeanDefinitionValueResolver解析属性值：

8、BeanWrapperImpl对Bean属性的依赖注入：

### 五、IoC容器的高级特性

1、介绍

2、Spring IoC容器的lazy-init属性实现预实例化：

(1) .refresh()

(2).finishBeanFactoryInitialization处理预实例化Bean：

(3) .DefaultListableBeanFactory对配置lazy-init属性单态Bean的预实例化：

3、FactoryBean的实现：

(1).`FactoryBean`的源码如下：

(2).` AbstractBeanFactory`的`getBean`方法调用`FactoryBean`：

(3)、`AbstractBeanFactory`生产Bean实例对象：

(4).工厂Bean的实现类getObject方法创建Bean实例对象：

4.BeanPostProcessor后置处理器的实现：

(1).BeanPostProcessor的源码如下：

(2).AbstractAutowireCapableBeanFactory类对容器生成的Bean添加后置处理器：

(3).initializeBean方法为容器产生的Bean实例对象添加BeanPostProcessor后置处理器：

(4).AdvisorAdapterRegistrationManager在Bean对象初始化后注册通知适配器：

5.Spring IoC容器autowiring实现原理：

(1). AbstractAutoWireCapableBeanFactory对Bean实例进行属性依赖注入：

(2).Spring IoC容器根据Bean名称或者类型进行autowiring自动依赖注入：

(3).DefaultSingletonBeanRegistry的registerDependentBean方法对属性注入：

## 一、什么是Ioc/DI？

​    IoC 容器：最主要是完成了完成对象的创建和依赖的管理注入等等。

先从我们自己设计这样一个视角来考虑：

所谓控制反转，就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现。那么必然的我们需要创建一个容器，同时需要一种描述来让容器知道需要创建的对象与对象的关系。这个描述最具体表现就是我们可配置的文件。

对象和对象关系怎么表示？

可以用 `xml`，`properties `文件等语义化配置文件表示。

描述对象关系的文件存放在哪里？

可能是 `classpath` ，` filesystem `，或者是 URL 网络资源，` servletContext `等。

回到正题，有了配置文件，还需要对配置文件解析。

不同的配置文件对对象的描述不一样，如标准的，自定义声明式的，如何统一？ 

在内部需要有一个统一的关于对象的定义，所有外部的描述都必须转化成统一的描述定义。

如何对不同的配置文件进行解析？

需要对不同的配置文件语法，采用不同的解析器

 

## 二、 Spring IOC体系结构？

(1) BeanFactory

​         Spring Bean的创建是典型的工厂模式，这一系列的Bean工厂，也即IOC容器为开发者管理对象间的依赖关系提供了很多便利和基础服务，在Spring中有许多的IOC容器的实现供用户选择和使用，其相互关系如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.151_3583.jpeg)

其中BeanFactory作为最顶层的一个接口类，它定义了IOC容器的基本功能规范，BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和AutowireCapableBeanFactory。但是从上图中我们可以发现最终的默认实现类是 DefaultListableBeanFactory，他实现了所有的接口。

那为何要定义这么多层次的接口呢？

查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。例如 ListableBeanFactory 接口表示这些 Bean 是可列表的，而 HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个Bean 有可能有父 Bean。AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为.

最基本的IOC容器接口BeanFactory

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.264_2114.jpeg)

在BeanFactory里只对IOC容器的基本行为作了定义，根本不关心你的bean是如何定义怎样加载的。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。

​      而要知道工厂是如何产生对象的，我们需要看具体的IOC容器实现，spring提供了许多IOC容器的实现。比如XmlBeanFactory，ClasspathXmlApplicationContext等。其中XmlBeanFactory就是针对最基本的ioc容器的实现，这个IOC容器可以读取XML文件定义的BeanDefinition（XML文件中对bean的描述）,如果说XmlBeanFactory是容器中的屌丝，ApplicationContext应该算容器中的高帅富.

​      ApplicationContext是Spring提供的一个高级的IoC容器，它除了能够提供IoC容器的基本功能外，还为用户提供了以下的附加服务。

从ApplicationContext接口的实现，我们看出其特点：

  1.  支持信息源，可以实现国际化。（实现MessageSource接口）

  2.  访问资源。(实现ResourcePatternResolver接口，这个后面要讲)

  3.  支持应用事件。(实现ApplicationEventPublisher接口)

(2) BeanDefinition

​         SpringIOC容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是以BeanDefinition来描述的，其继承体系如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.776_20759.jpeg)

Bean 的解析过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。Bean 的解析主要就是对 Spring 配置文件的解析。这个解析过程主要通过下图中的类完成：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.153_1679.jpeg)

## 三、IoC容器的初始化？

​      IoC容器的初始化包括BeanDefinition的Resource定位、载入和注册这三个基本的过程。我们以ApplicationContext为例讲解，ApplicationContext系列容器也许是我们最熟悉的，因为web项目中使用的XmlWebApplicationContext就属于这个继承体系，还有ClasspathXmlApplicationContext等，其继承体系如下图所示：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.765_80967.jpeg)

ApplicationContext允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。对于bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的bean定义环境。

下面我们分别简单地演示一下两种ioc容器的创建过程

**1、XmlBeanFactory(屌丝IOC)的整个流程**

通过XmlBeanFactory的源码，我们可以发现:

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.131_42763.png)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.764_61523.png)

通过前面的源码，`this.reader = new XmlBeanDefinitionReader(this); `中其中this 传的是factory对象

**2、FileSystemXmlApplicationContext 的IOC容器流程**

**2.1高富帅IOC解剖**

**1 、ApplicationContext =new FileSystemXmlApplicationContext(xmlPath);**

先看其构造函数：

 调用构造函数：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.457_39738.png)

实际调用

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.109_68809.png)

**2、设置资源加载器和资源定位**

通过分析`FileSystemXmlApplicationContext`的源代码可以知道，在创建`FileSystemXmlApplicationContext`容器时，构造方法做以下两项重要工作：

首先，调用父类容器的构造方法`(super(parent)方法)`为容器设置好Bean资源加载器。

然后，再调用父类`AbstractRefreshableConfigApplicationContext`的`setConfigLocations(configLocations)`方法设置Bean定义资源文件的定位路径。

通过追踪`FileSystemXmlApplicationContext`的继承体系，发现其父类的父类`AbstractApplicationContext`中初始化IoC容器所做的主要源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.765_65979.jpeg)

`AbstractApplicationContext`构造方法中调用`PathMatchingResourcePatternResolver`的构造方法创建Spring资源加载器：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.127_67970.png)

在设置容器的资源加载器之后，接下来`FileSystemXmlApplicationContet`执行`setConfigLocations`方法通过调用其父类`AbstractRefreshableConfigApplicationContext`的方法进行对Bean定义资源文件的定位，该方法的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.078_82410.png)

通过这两个方法的源码我们可以看出，我们既可以使用一个字符串来配置多个Spring Bean定义资源文件，也可以使用字符串数组，即下面两种方式都是可以的：

a.    `ClasspathResource res = new ClasspathResource(“a.xml,b.xml,……”);`

多个资源文件路径之间可以是用” ,; /t/n”等分隔。

b. `   ClasspathResource res = new ClasspathResource(newString[]{“a.xml”,”b.xml”,……});`

至此，Spring IoC容器在初始化时将配置的Bean定义资源文件定位为Spring封装的Resource。

**3、AbstractApplicationContext的refresh函数载入Bean定义过程：**

Spring IoC容器对Bean定义资源的载入是从refresh()函数开始的，refresh()是一个模板方法，refresh()方法的作用是：在创建IoC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IoC容器。refresh的作用类似于对IoC容器的重启，在新建立好的容器中对容器进行初始化，对Bean定义资源进行载入

`FileSystemXmlApplicationContext`通过调用其父类`AbstractApplicationContext`的`refresh()`函数启动整个IoC容器对Bean定义的载入过程：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.410_97885.jpeg)

refresh()方法主要为IoC容器Bean的生命周期管理提供条件，Spring IoC容器载入Bean定义资源文件从其子类容器的`refreshBeanFactory()`方法启动，所以整个refresh()中`“ConfigurableListableBeanFactory beanFactory =obtainFreshBeanFactory();”`这句以后代码的都是注册容器的信息源和生命周期事件，载入过程就是从这句代码启动。

 **refresh()方法的作用是：**在创建IoC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IoC容器。refresh的作用类似于对IoC容器的重启，在新建立好的容器中对容器进行初始化，对Bean定义资源进行载入

`AbstractApplicationContext`的`obtainFreshBeanFactory()`方法调用子类容器的`refreshBeanFactory()`方法，启动容器载入Bean定义资源文件的过程，代码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.762_86254.png)

**AbstractApplicationContext子类的refreshBeanFactory()方法：**

 `  AbstractApplicationContext`类中只抽象定义了`refreshBeanFactory()`方法，容器真正调用的是其子类`AbstractRefreshableApplicationContext`实现的    `refreshBeanFactory()`方法，方法的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.503_10373.jpeg)

在这个方法中，先判断BeanFactory是否存在，如果存在则先销毁beans并关闭`beanFactory`，接着创建`DefaultListableBeanFactory`，并调用`loadBeanDefinitions(beanFactory)`装载bean定义。

**5、AbstractRefreshableApplicationContext子类的loadBeanDefinitions方法：**

`AbstractRefreshableApplicationContext`中只定义了抽象的`loadBeanDefinitions`方法，容器真正调用的是其子类`AbstractXmlApplicationContext`对该方法的实现，`AbstractXmlApplicationContext`的主要源码如下：

`loadBeanDefinitions`方法同样是抽象方法，是由其子类实现的，也即在`AbstractXmlApplicationContext`中。

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.201_62277.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.083_88912.jpeg)

Xml Bean读取器(XmlBeanDefinitionReader)调用其父类`AbstractBeanDefinitionReader`的 `reader.loadBeanDefinitions`方法读取Bean定义资源。

由于我们使用`FileSystemXmlApplicationContext`作为例子分析，因此`getConfigResources`的返回值为null，因此程序执行`reader.loadBeanDefinitions(configLocations)`分支。

**6、AbstractBeanDefinitionReader读取Bean定义资源：**

`AbstractBeanDefinitionReader`的`loadBeanDefinitions`方法源码如下：

可以到`org.springframework.beans.factory.support`看一下`BeanDefinitionReader`的结构

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.460_72474.png)

在其抽象父类`AbstractBeanDefinitionReader`中定义了载入过程

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.107_36323.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.777_24355.jpeg)

 `loadBeanDefinitions(Resource...resources)`方法和上面分析的3个方法类似，同样也是调用`XmlBeanDefinitionReader`的`loadBeanDefinitions`方法。

从对`AbstractBeanDefinitionReader`的`loadBeanDefinitions`方法源码分析可以看出该方法做了以下两件事：

首先，调用资源加载器的获取资源方法`resourceLoader.getResource(location)`，获取到要加载的资源。

其次，真正执行加载功能是其子类`XmlBeanDefinitionReader的loadBeanDefinitions`方法。

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.088_33508.png) ![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.819_65734.png)

看到第8、16行，结合上面的`ResourceLoader`与`ApplicationContext`的继承关系图，可以知道此时调用的是`DefaultResourceLoader`中的`getSource()`方法定位`Resource`，因为`FileSystemXmlApplicationContext`本身就是`DefaultResourceLoader`的实现类，所以此时又回到了`FileSystemXmlApplicationContext`中来。

**7、资源加载器获取要读入的资源：**

`XmlBeanDefinitionReader`通过调用其父类`DefaultResourceLoader的getResource`方法获取要加载的资源，其源码如下

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.762_56970.jpeg)

`FileSystemXmlApplicationContext`容器提供了`getResourceByPath`方法的实现，就是为了处理既不是`classpath`标识，又不是URL标识的`Resource`定位这种情况。

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.115_55463.png)

这样代码就回到了` FileSystemXmlApplicationContext `中来，他提供了`FileSystemResource `来完成从文件系统得到配置文件的资源定义。

这样，就可以从文件系统路径上对IOC 配置文件进行加载 - 当然我们可以按照这个逻辑从任何地方加载，在Spring 中我们看到它提供 的各种资源抽象，比如`ClassPathResource`, `URLResource`、`FileSystemResource `等来供我们使用。上面我们看到的是定位Resource 的一个过程，而这只是加载过程的一部分.

**8、XmlBeanDefinitionReader加载Bean定义资源：**

​     Bean定义的Resource得到了

继续回到`XmlBeanDefinitionReader`的`loadBeanDefinitions(Resource …)`方法看到代表bean文件的资源定义以后的载入过程。

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.413_13729.jpeg)

通过源码分析，载入Bean定义资源文件的最后一步是将Bean定义资源转换为`Document`对象，该过程由documentLoader实现

**9、DocumentLoader将Bean定义资源转换为Document对象：**

`DocumentLoader`将Bean定义资源转换成`Document`对象的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.510_78849.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.199_29831.png)

该解析过程调用JavaEE标准的JAXP标准进行处理。

至此Spring IoC容器根据定位的Bean定义资源文件，将其加载读入并转换成为Document对象过程完成。

接下来我们要继续分析Spring IoC容器将载入的Bean定义资源文件转换为Document对象之后，是如何将其解析为Spring IoC管理的Bean对象并将其注册到容器中的。

**10、XmlBeanDefinitionReader解析载入的Bean定义资源文件：**

` XmlBeanDefinitionReader`类中的`doLoadBeanDefinitions`方法是从特定XML文件中实际载入Bean定义资源的方法，该方法在载入Bean定义资源之后将其转换为Document对象，接下来调用`registerBeanDefinitions`启动Spring IoC容器对Bean定义的解析过程，`registerBeanDefinitions`方法源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.105_76186.jpeg)

Bean定义资源的载入解析分为以下两个过程：

首先，通过调用XML解析器将Bean定义资源文件转换得到Document对象，但是这些Document对象并没有按照Spring的Bean规则进行解析。这一步是载入的过程

其次，在完成通用的XML解析之后，按照Spring的Bean规则对`Document`对象进行解析。

按照Spring的Bean规则对Document对象解析的过程是在接口`BeanDefinitionDocumentReader`的实现类`DefaultBeanDefinitionDocumentReader`中实现的。

**11、DefaultBeanDefinitionDocumentReader对Bean定义的Document对象解析：**

`BeanDefinitionDocumentReader`接口通过`registerBeanDefinitions`方法调用其实现类`DefaultBeanDefinitionDocumentReader`对`Document`对象进行解析，解析的代码如下：

 ![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.829_37588.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.504_78520.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.082_24662.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.529_53513.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.802_98199.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.088_6875.jpeg)

通过上述Spring IoC容器对载入的Bean定义Document解析可以看出，我们使用Spring时，在Spring配置文件中可以使用`<Import>`元素来导入IoC容器所需要的其他资源，`Spring IoC`容器在解析时会首先将指定导入的资源加载进容器中。使用`<Ailas>`别名时，`Spring IoC`容器首先将别名元素所定义的别名注册到容器中。

对于既不是`<Import>`元素，又不是`<Alias>`元素的元素，即Spring配置文件中普通的`<Bean>`元素的解析由`BeanDefinitionParserDelegate`类的`parseBeanDefinitionElement`方法来实现。

**12、BeanDefinitionParserDelegate解析Bean定义资源文件中的`<Bean>`元素：**

Bean定义资源文件中的`<Import>`和`<Alias>`元素解析在`DefaultBeanDefinitionDocumentReader`中已经完成，对Bean定义资源文件中使用最多的`<Bean>`元素交由`BeanDefinitionParserDelegate`来解析，其解析实现的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.107_19257.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.412_96252.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.108_63993.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.458_6230.png)

只要使用过Spring，对Spring配置文件比较熟悉的人，通过对上述源码的分析，就会明白我们在Spring配置文件中`<Bean>`元素的中配置的属性就是通过该方法解析和设置到Bean中去的。

**注意：**在解析`<Bean>`元素过程中没有创建和实例化Bean对象，只是创建了Bean对象的定义类`BeanDefinition`，将`<Bean>`元素中的配置信息设置到`BeanDefinition`中作为记录，当依赖注入时才使用这些记录信息创建和实例化具体的Bean对象。

上面方法中一些对一些配置如元信息(meta)、qualifier等的解析，我们在Spring中配置时使用的也不多，我们在使用Spring的`<Bean>`元素时，配置最多的是`<property>`属性，因此我们下面继续分析源码，了解Bean的属性在解析时是如何设置的。

**13、BeanDefinitionParserDelegate解析`<property>`元素：**

`BeanDefinitionParserDelegate`在解析`<Bean>`调用parsePropertyElements方法解析`<Bean>`元素中的`<property>`属性子元素，解析源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.804_12880.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.213_55887.jpeg)![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.077_32523.jpeg)

通过对上述源码的分析，我们可以了解在Spring配置文件中，`<Bean>`元素中`<property>`元素的相关配置是如何处理的：

**a.** ref被封装为指向依赖对象一个引用。

**b.**value配置都会封装成一个字符串类型的对象。

**c.**ref和value都通过“解析的数据类型属性值.`setSource(extractSource(ele));`”方法将属性值/引用与所引用的属性关联起来。

在方法的最后对于`<property>`元素的子元素通过`parsePropertySubElement `方法解析，我们继续分析该方法的源码，了解其解析过程。

**14、解析`<property>`元素的子元素：**

在`BeanDefinitionParserDelegate`类中的`parsePropertySubElement`方法对`<property>`中的子元素解析，源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.804_64692.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.803_21541.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.459_20480.png)

通过上述源码分析，我们明白了在Spring配置文件中，对`<property>`元素中配置的`Array、List、Set、Map、Prop`等各种集合子元素的都通过上述方法解析，生成对应的数据对象，比如`ManagedList、ManagedArray、ManagedSet`等，这些Managed类是Spring对象`BeanDefiniton`的数据封装，对集合数据类型的具体解析有各自的解析方法实现，解析方法的命名非常规范，一目了然，我们对`<list>`集合元素的解析方法进行源码分析，了解其实现过程。

**15、解析`<list>`子元素：**

在`BeanDefinitionParserDelegate`类中的`parseListElement`方法就是具体实现解析`<property>`元素中的`<list>`集合子元素，源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.413_22633.jpeg)

经过对Spring Bean定义资源文件转换的Document对象中的元素层层解析，Spring IoC现在已经将XML形式定义的Bean定义资源文件转换为Spring IoC所识别的数据结构——BeanDefinition，它是Bean定义资源文件中配置的POJO对象在Spring IoC容器中的映射，我们可以通过AbstractBeanDefinition为入口，荣IoC容器进行索引、查询和操作。

通过Spring IoC容器对Bean定义资源的解析后，IoC容器大致完成了管理Bean对象的准备工作，即初始化过程，但是最为重要的依赖注入还没有发生，现在在IoC容器中BeanDefinition存储的只是一些静态信息，接下来需要向容器注册Bean定义信息才能全部完成IoC容器的初始化过程

**16、解析过后的BeanDefinition在IoC容器中的注册：**

让我们继续跟踪程序的执行顺序，接下来会到我们第3步中分析`DefaultBeanDefinitionDocumentReader`对Bean定义转换的Document对象解析的流程中，在其`parseDefaultElement`方法中完成对Document对象的解析后得到封装`BeanDefinition`的`BeanDefinitionHold`对象，然后调用`BeanDefinitionReaderUtils`的`registerBeanDefinition`方法向IoC容器注册解析的Bean，`BeanDefinitionReaderUtils`的注册的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.155_88960.png)

当调用`BeanDefinitionReaderUtils`向IoC容器注册解析的`BeanDefinition`时，真正完成注册功能的是`DefaultListableBeanFactory`。

**17、`DefaultListableBeanFactory`向IoC容器注册解析后的BeanDefinition：**

`DefaultListableBeanFactory`中使用一个HashMap的集合对象存放IoC容器中注册解析的`BeanDefinition`，向IoC容器注册的主要源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.080_35496.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.819_13611.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.460_41098.png)

至此，Bean定义资源文件中配置的Bean被解析过后，已经注册到IoC容器中，被容器管理起来，真正完成了IoC容器初始化所做的全部工作。现  在IoC容器中已经建立了整个Bean的配置信息，这些BeanDefinition信息已经可以使用，并且可以被检索，IoC容器的作用就是对这些注册的Bean定义信息进行处理和维护。这些的注册的Bean定义信息是IoC容器控制反转的基础，正是有了这些注册的数据，容器才可以进行依赖注入。

**总结：**

**现在通过上面的代码，总结一下IOC容器初始化的基本步骤：**

1.初始化的入口在容器实现中的 refresh()调用来完成

2.对 bean 定义载入 IOC 容器使用的方法是 loadBeanDefinition,其中的大致过程如下：通过 ResourceLoader 来完成资源文件位置的定位，DefaultResourceLoader 是默认的实现，同时上下文本身就给出了 ResourceLoader 的实现，可以从类路径，文件系统, URL 等方式来定为资源位置。如果是 XmlBeanFactory作为 IOC 容器，那么需要为它指定 bean 定义的资源，也就是说 bean 定义文件时通过抽象成 Resource 来被 IOC 容器处理的，容器通过 BeanDefinitionReader来完成定义信息的解析和 Bean 信息的注册,往往使用的是XmlBeanDefinitionReader 来解析 bean 的 xml 定义文件 - 实际的处理过程是委托给 BeanDefinitionParserDelegate 来完成的，从而得到 bean 的定义信息，这些信息在 Spring 中使用 BeanDefinition 对象来表示 - 这个名字可以让我们想到loadBeanDefinition,RegisterBeanDefinition  这些相关的方法 - 他们都是为处理 BeanDefinitin 服务的， 容器解析得到 BeanDefinitionIoC 以后，需要把它在 IOC 容器中注册，这由 IOC 实现 BeanDefinitionRegistry 接口来实现。注册过程就是在 IOC 容器内部维护的一个HashMap 来保存得到的 BeanDefinition 的过程。这个 HashMap 是 IoC 容器持有 bean 信息的场所，以后对 bean 的操作都是围绕这个HashMap 来实现的.

3.然后我们就可以通过 `BeanFactory` 和 `ApplicationContext `来享受到`Spring IOC `的服务了,在使用 IOC 容器的时候，我们注意到除了少量粘合代码，绝大多数以正确 IoC 风格编写的应用程序代码完全不用关心如何到达工厂，因为容器将把这些对象与容器管理的其他对象钩在一起。基本的策略是把工厂放到已知的地方，最好是放在对预期使用的上下文有意义的地方，以及代码将实际需要访问工厂的地方。 Spring 本身提供了对声明式载入 web 应用程序用法的应用程序上下文,并将其存储在`ServletContext `中的框架实现。具体可以参见以后的文章 

**在使用 Spring IOC 容器的时候我们还需要区别两个概念:**

`Beanfactory` 和 `Factory bean`，其中 BeanFactory 指的是 IOC 容器的编程抽象，比如 `ApplicationContext`， `XmlBeanFactory `等，这些都是 IOC 容器的具体表现，需要使用什么样的容器由客户决定,但 Spring 为我们提供了丰富的选择。 `FactoryBean `只是一个可以在 IOC而容器中被管理的一个 bean,是对各种处理过程和资源使用的抽象,Factory bean 在需要时产生另一个对象，而不返回 FactoryBean本身,我们可以把它看成是一个抽象工厂，对它的调用返回的是工厂生产的产品。所有的 Factory bean 都实现特殊的`org.springframework.beans.factory.FactoryBean `接口，当使用容器中 factory bean 的时候，该容器不会返回 `factory bean` 本身,而是返回其生成的对象。Spring 包括了大部分的通用资源和服务访问抽象的 Factory bean 的实现，其中包括:对 JNDI 查询的处理，对代理对象的处理，对事务性代理的处理，对 RMI 代理的处理等，这些我们都可以看成是具体的工厂,看成是SPRING 为我们建立好的工厂。也就是说 Spring 通过使用抽象工厂模式为我们准备了一系列工厂来生产一些特定的对象,免除我们手工重复的工作，我们要使用时只需要在 IOC 容器里配置好就能很方便的使用了

## 四、IOC容器的依赖注入

**1、依赖注入发生的时间**

当`Spring IoC`容器完成了Bean定义资源的定位、载入和解析注册以后，IoC容器中已经管理类Bean定义的相关数据，但是此时IoC容器还没有对所管理的Bean进行依赖注入，依赖注入在以下两种情况发生：

(1).用户第一次通过getBean方法向IoC容索要Bean时，IoC容器触发依赖注入。

(2).当用户在Bean定义资源中为`<Bean>`元素配置了`lazy-init`属性，即让容器在解析注册Bean定义时进行预实例化，触发依赖注入。

`BeanFactory`接口定义了`Spring IoC`容器的基本功能规范，是Spring IoC容器所应遵守的最底层和最基本的编程规范。`BeanFactory`接口中定义了几个getBean方法，就是用户向IoC容器索取管理的Bean的方法，我们通过分析其子类的具体实现，理解`Spring IoC`容器在用户索取Bean时如何完成依赖注入。

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.458_85138.jpeg)

 

在BeanFactory中我们看到`getBean（String…）`函数，它的具体实现在`AbstractBeanFactory`中

**2、AbstractBeanFactory通过getBean向IoC容器获取被管理的Bean：**

AbstractBeanFactory的getBean相关方法的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.460_21306.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.411_44006.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.509_87392.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.814_17715.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.106_7947.jpeg)

通过上面对向IoC容器获取Bean方法的分析，我们可以看到在Spring中，如果Bean定义的单态模式(Singleton)，则容器在创建之前先从缓存中查找，以确保整个容器中只存在一个实例对象。如果Bean定义的是原型模式(Prototype)，则容器每次都会创建一个新的实例对象。除此之外，Bean定义还可以扩展为指定其生命周期范围。

上面的源码只是定义了根据Bean定义的模式，采取的不同创建Bean实例对象的策略，具体的Bean实例对象的创建过程由实现了ObejctFactory接口的匿名内部类的createBean方法完成，ObejctFactory使用委派模式，具体的Bean实例创建过程交由其实现类`AbstractAutowireCapableBeanFactory`完成，我们继续分析`AbstractAutowireCapableBeanFactory`的createBean方法的源码，理解其创建Bean实例的具体实现过程。

**3、AbstractAutowireCapableBeanFactory创建Bean实例对象：**

`AbstractAutowireCapableBeanFactory`类实现了`ObejctFactory`接口，创建容器指定的Bean实例对象，同时还对创建的Bean实例对象进行初始化处理。其创建Bean实例对象的方法源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.824_99538.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.817_83236.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.824_43340.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.821_76347.jpeg)

**通过对方法源码的分析，我们看到具体的依赖注入实现在以下两个方法中：**

(1).`createBeanInstance`：生成Bean所包含的java对象实例。

(2).`populateBean` ：对Bean属性的依赖注入进行处理。

下面继续分析这两个方法的代码实现。

**4、createBeanInstance方法创建Bean的java实例对象：**

在`createBeanInstance`方法中，根据指定的初始化策略，使用静态工厂、工厂方法或者容器的自动装配特性生成java实例对象，创建对象的源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.813_55982.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.107_14446.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.083_16242.png)

经过对上面的代码分析，我们可以看出，对使用工厂方法和自动装配特性的Bean的实例化相当比较清楚，调用相应的工厂方法或者参数匹配的构造方法即可完成实例化对象的工作，但是对于我们最常使用的默认无参构造方法就需要使用相应的初始化策略(JDK的反射机制或者CGLIB)来进行初始化了，在方法`getInstantiationStrategy().instantiate`中就具体实现类使用初始策略实例化对象。

**5、SimpleInstantiationStrategy类使用默认的无参构造方法创建Bean实例化对象：**

在使用默认的无参构造方法创建Bean的实例化对象时，方法`getInstantiationStrategy().instantiate调`用了`SimpleInstantiationStrategy`类中的实例化Bean的方法，其源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.804_20237.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.458_65695.png)

通过上面的代码分析，我们看到了如果Bean有方法被覆盖了，则使用JDK的反射机制进行实例化，否则，使用CGLIB进行实例化。

`instantiateWithMethodInjection`方法调用`SimpleInstantiationStrategy`的子类`CglibSubclassingInstantiationStrategy`使用CGLIB来进行初始化，其源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.461_20630.png)

CGLIB是一个常用的字节码生成器的类库，它提供了一系列API实现java字节码的生成和转换功能。我们在学习JDK的动态代理时都知道，JDK的动态代理只能针对接口，如果一个类没有实现任何接口，要对其进行动态代理只能使用CGLIB。

**6、populateBean方法对Bean属性的依赖注入：**

**在第3步的分析中我们已经了解到Bean的依赖注入分为以下两个过程：**

(1).`createBeanInstance`：生成Bean所包含的java对象实例。

(2).`populateBean `：对Bean属性的依赖注入进行处理。

第4、5步中我们已经分析了容器初始化生成Bean所包含的Java实例对象的过程，现在我们继续分析生成对象后，Spring IoC容器是如何将Bean的属性依赖关系注入Bean实例对象中并设置好的，属性依赖注入的代码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.108_14890.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.511_11115.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.251_11836.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.504_60408.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.533_82177.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.078_78151.png)

分析上述代码，我们可以看出，对属性的注入过程分以下两种情况：

(1).属性值类型不需要转换时，不需要解析属性值，直接准备进行依赖注入。

(2).属性值需要进行类型转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

对属性值的解析是在`BeanDefinitionValueResolver`类中的`resolveValueIfNecessary`方法中进行的，对属性值的依赖注入是通过`bw.setPropertyValues`方法实现的，在分析属性值的依赖注入之前，我们先分析一下对属性值的解析过程。

**7、BeanDefinitionValueResolver解析属性值：**

当容器在对属性进行依赖注入时，如果发现属性值需要进行类型转换，如属性值是容器中另一个Bean实例对象的引用，则容器首先需要根据属性值解析出所引用的对象，然后才能将该引用对象注入到目标实例对象的属性上去，对属性进行解析的由`resolveValueIfNecessary`方法实现，其源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.802_16567.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.552_29126.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.239_18652.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.410_25089.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.203_56396.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.457_7182.png)

通过上面的代码分析，我们明白了Spring是如何将引用类型，内部类以及集合类型等属性进行解析的，属性值解析完成后就可以进行依赖注入了，依赖注入的过程就是Bean对象实例设置到它所依赖的Bean对象属性上去，在第7步中我们已经说过，依赖注入是通过`bw.setPropertyValues`方法实现的，该方法也使用了委托模式，在`BeanWrapper`接口中至少定义了方法声明，依赖注入的具体实现交由其实现类`BeanWrapperImpl`来完成，下面我们就分析依`BeanWrapperImpl`中赖注入相关的源码。

**8、BeanWrapperImpl对Bean属性的依赖注入：**

`BeanWrapperImpl`类主要是对容器中完成初始化的Bean实例对象进行属性的依赖注入，即把Bean对象设置到它所依赖的另一个Bean的属性中去，依赖注入的相关源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.108_27714.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.518_41516.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.775_81884.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.411_69638.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.211_29739.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.803_27700.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.506_33287.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.411_4426.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.122_26772.png)

通过对上面注入依赖代码的分析，我们已经明白了Spring IoC容器是如何将属性的值注入到Bean实例对象中去的：

(1).对于集合类型的属性，将其属性值解析为目标类型的集合后直接赋值给属性。

(2).对于非集合类型的属性，大量使用了JDK的反射和内省机制，通过属性的getter方法(reader method)获取指定属性注入以前的值，同时调用属性的setter方法(writer method)为属性设置注入后的值。看到这里相信很多人都明白了Spring的setter注入原理。

至此Spring IoC容器对Bean定义资源文件的定位，载入、解析和依赖注入已经全部分析完毕，现在Spring IoC容器中管理了一系列靠依赖关系联系起来的Bean，程序不需要应用自己手动创建所需的对象，Spring IoC容器会在我们使用的时候自动为我们创建，并且为我们注入好相关的依赖，这就是Spring核心功能的控制反转和依赖注入的相关功能。

## 五、IoC容器的高级特性

**1、介绍**

​      通过前面4篇文章对Spring IoC容器的源码分析，我们已经基本上了解了Spring IoC容器对Bean定义资源的定位、读入和解析过程，同时也清楚了当用户通过getBean方法向IoC容器获取被管理的Bean时，IoC容器对Bean进行的初始化和依赖注入过程，这些是Spring IoC容器的基本功能特性。Spring IoC容器还有一些高级特性，如使用`lazy-init`属性对Bean预初始化、`FactoryBean`产生或者修饰Bean对象的生成、IoC容器初始化Bean过程中使用`BeanPostProcessor`后置处理器对Bean声明周期事件管理和IoC容器的`autowiring`自动装配功能等。

**2、Spring IoC容器的lazy-init属性实现预实例化：**

​      通过前面我们对IoC容器的实现和工作原理分析，我们知道IoC容器的初始化过程就是对Bean定义资源的定位、载入和注册，此时容器对Bean的依赖注入并没有发生，依赖注入主要是在应用程序第一次向容器索取Bean时，通过getBean方法的调用完成。

当Bean定义资源的`<Bean>`元素中配置了`lazy-init`属性时，容器将会在初始化的时候对所配置的Bean进行预实例化，Bean的依赖注入在容器初始化的时候就已经完成。这样，当应用程序第一次向容器索取被管理的Bean时，就不用再初始化和对Bean进行依赖注入了，直接从容器中获取已经完成依赖注入的现成Bean，可以提高应用第一次向容器获取Bean的性能。

下面我们通过代码分析容器预实例化的实现过程：

**(1).refresh()**

先从IoC容器的初始会过程开始，通过前面文章分析，我们知道IoC容器读入已经定位的Bean定义资源是从refresh方法开始的，我们首先从`AbstractApplicationContext`类的`refresh`方法入手分析，源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.109_48929.jpeg)

在refresh方法中`ConfigurableListableBeanFactorybeanFactory = obtainFreshBeanFactory()`;启动了Bean定义资源的载入、注册过程，而`finishBeanFactoryInitialization`方法是对注册后的Bean定义中的预实例化(`lazy-init=false`，Spring默认就是预实例化，即为true)的Bean进行处理的地方。

**(2).finishBeanFactoryInitialization处理预实例化Bean：**

当Bean定义资源被载入IoC容器之后，容器将Bean定义资源解析为容器内部的数据结构`BeanDefinition`注册到容器中，`AbstractApplicationContext`类中的`finishBeanFactoryInitialization`方法对配置了预实例化属性的Bean进行预初始化过程，源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.803_38779.jpeg)

`ConfigurableListableBeanFactory`是一个接口，其`preInstantiateSingletons`方法由其子类`DefaultListableBeanFactory`提供。  

**(3)、DefaultListableBeanFactory对配置lazy-init属性单态Bean的预实例化：**

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.514_77531.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.113_23436.png)

通过对lazy-init处理源码的分析，我们可以看出，如果设置了lazy-init属性，则容器在完成Bean定义的注册之后，会通过getBean方法，触发对指定Bean的初始化和依赖注入过程，这样当应用第一次向容器索取所需的Bean时，容器不再需要对Bean进行初始化和依赖注入，直接从已经完成实例化和依赖注入的Bean中取一个线程的Bean，这样就提高了第一次获取Bean的性能。

**3、FactoryBean的实现：**

在Spring中，有两个很容易混淆的类：`BeanFactory`和`FactoryBean`。
`BeanFactory`：Bean工厂，是一个工厂(Factory)，我们Spring IoC容器的最顶层接口就是这个BeanFactory，它的作用是管理Bean，即实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

FactoryBean：工厂Bean，是一个Bean，作用是产生其他bean实例。通常情况下，这种bean没有什么特别的要求，仅需要提供一个工厂方法，该方法用来返回其他bean实例。通常情况下，bean无须自己实现工厂模式，Spring容器担任工厂角色；但少数情况下，容器中的bean本身就是工厂，其作用是产生其它bean实例。

当用户使用容器本身时，可以使用转义字符”&”来得到FactoryBean本身，以区别通过FactoryBean产生的实例对象和FactoryBean对象本身。在BeanFactory中通过如下代码定义了该转义字符：

StringFACTORY_BEAN_PREFIX = "&";

如果myJndiObject是一个FactoryBean，则使用&myJndiObject得到的是myJndiObject对象，而不是myJndiObject产生出来的对象。

**(1).FactoryBean的源码如下：**

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.459_70467.png)

**(2). AbstractBeanFactory的getBean方法调用FactoryBean：**

在前面我们分析`Spring Ioc`容器实例化Bean并进行依赖注入过程的源码时，提到在`getBean`方法触发容器实例化Bean的时候会调用`AbstractBeanFactory`的`doGetBean`方法来进行实例化的过程，源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.412_68255.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.813_44603.jpeg)

在上面获取给定Bean的实例对象的`getObjectForBeanInstance`方法中，会调用`FactoryBeanRegistrySupport`类的`getObjectFromFactoryBean`方法，该方法实现了Bean工厂生产Bean实例对象。

`Dereference`(解引用)：一个在C/C++中应用比较多的术语，在C++中，”*”是解引用符号，而”&”是引用符号，解引用是指变量指向的是所引用对象的本身数据，而不是引用对象的内存地址。

**(3)、AbstractBeanFactory生产Bean实例对象：**

`AbstractBeanFactory`类中生产Bean实例对象的主要源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.805_61083.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.106_73574.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.762_94247.png)

从上面的源码分析中，我们可以看出，`BeanFactory`接口调用其实现类的`getObject`方法来实现创建Bean实例对象的功能。

**(4).工厂Bean的实现类getObject方法创建Bean实例对象：**

`FactoryBean`的实现类有非常多，比如：`Proxy、RMI、JNDI、ServletContextFactoryBean`等等，`FactoryBean`接口为Spring容器提供了一个很好的封装机制，具体的getObject有不同的实现类根据不同的实现策略来具体提供，我们分析一个最简单的`AnnotationTestFactoryBean`的实现源码：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.764_9593.png)

其他的Proxy，RMI，JNDI等等，都是根据相应的策略提供getObject的实现。这里不做一一分析，这已经不是Spring的核心功能，有需要的时候再去深入研究。

**4.BeanPostProcessor后置处理器的实现：**

`BeanPostProcessor`后置处理器是Spring IoC容器经常使用到的一个特性，这个Bean后置处理器是一个监听器，可以监听容器触发的Bean声明周期事件。后置处理器向容器注册以后，容器中管理的Bean就具备了接收IoC容器事件回调的能力。

`BeanPostProcessor`的使用非常简单，只需要提供一个实现接口`BeanPostProcessor`的实现类，然后在Bean的配置文件中设置即可。

**(1).BeanPostProcessor的源码如下：**

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.085_46303.png)

这两个回调的入口都是和容器管理的Bean的生命周期事件紧密相关，可以为用户提供在`Spring IoC`容器初始化Bean过程中自定义的处理操作。

**(2).AbstractAutowireCapableBeanFactory类对容器生成的Bean添加后置处理器：**

BeanPostProcessor后置处理器的调用发生在`Spring IoC`容器完成对Bean实例对象的创建和属性的依赖注入完成之后，在对Spring依赖注入的源码分析过程中我们知道，当应用程序第一次调用getBean方法(lazy-init预实例化除外)向`Spring IoC`容器索取指定Bean时触发`Spring IoC`容器创建Bean实例对象并进行依赖注入的过程，其中真正实现创建Bean对象并进行依赖注入的方法是`AbstractAutowireCapableBeanFactory`类的`doCreateBean`方法，主要源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.773_45905.png)

从上面的代码中我们知道，为Bean实例对象添加`BeanPostProcessor`后置处理器的入口的是`initializeBean`方法。

**(3).initializeBean方法为容器产生的Bean实例对象添加BeanPostProcessor后置处理器：**

同样在`AbstractAutowireCapableBeanFactory`类中，`initializeBean`方法实现为容器创建的Bean实例对象添加`BeanPostProcessor`后置处理器，源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.804_73989.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.813_83392.jpeg)

`BeanPostProcessor`是一个接口，其初始化前的操作方法和初始化后的操作方法均委托其实现子类来实现，在Spring中，`BeanPostProcessor`的实现子类非常的多，分别完成不同的操作，如：AOP面向切面编程的注册通知适配器、Bean对象的数据校验、Bean继承属性/方法的合并等等，我们以最简单的AOP切面织入来简单了解其主要的功能。

**(4).AdvisorAdapterRegistrationManager在Bean对象初始化后注册通知适配器：**

`AdvisorAdapterRegistrationManager`是`BeanPostProcessor`的一个实现类，其主要的作用为容器中管理的Bean注册一个面向切面编程的通知适配器，以便在Spring容器为所管理的Bean进行面向切面编程时提供方便，其源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.088_53231.jpeg)

其他的`BeanPostProcessor`接口实现类的也类似，都是对Bean对象使用到的一些特性进行处理，或者向IoC容器中注册，为创建的Bean实例对象做一些自定义的功能增加，这些操作是容器初始化Bean时自动触发的，不需要认为的干预。

**5.Spring IoC容器autowiring实现原理：**

Spring IoC容器提供了两种管理Bean依赖关系的方式：

**a.** 显式管理：通过`BeanDefinition`的属性值和构造方法实现Bean依赖关系管理。

**b.** `autowiring`：`Spring IoC`容器的依赖自动装配功能，不需要对Bean属性的依赖关系做显式的声明，只需要在配置好`autowiring`属性，IoC容器会自动使用反射查找属性的类型和名称，然后基于属性的类型或者名称来自动匹配容器中管理的Bean，从而自动地完成依赖注入。

通过对`autowiring`自动装配特性的理解，我们知道容器对Bean的自动装配发生在容器对Bean依赖注入的过程中。在前面对Spring IoC容器的依赖注入过程源码分析中，我们已经知道了容器对Bean实例对象的属性注入的处理发生在`AbstractAutoWireCapableBeanFactory`类中的`populateBean`方法中，我们通过程序流程分析`autowiring`的实现原理：

**(1). AbstractAutoWireCapableBeanFactory对Bean实例进行属性依赖注入：**

应用第一次通过getBean方法(配置了`lazy-init`预实例化属性的除外)向IoC容器索取Bean时，容器创建Bean实例对象，并且对Bean实例对象进行属性依赖注入，`AbstractAutoWireCapableBeanFactory`的`populateBean`方法就是实现Bean属性依赖注入的功能，其主要源码如下：

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.110_44908.jpeg)

**(2).Spring IoC容器根据Bean名称或者类型进行autowiring自动依赖注入：**

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:05.235_96170.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.107_66141.jpeg)

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:06.805_33382.png)

通过上面的源码分析，我们可以看出来通过属性名进行自动依赖注入的相对比通过属性类型进行自动依赖注入要稍微简单一些，但是真正实现属性注入的是`DefaultSingletonBeanRegistry`类的`registerDependentBean`方法。

**(3).DefaultSingletonBeanRegistry的registerDependentBean方法对属性注入：**

![img](https://notecdn.yiban.io/cloud_res/479621162/imgs/21-6-4_13:53:07.412_88468.jpeg)

通过对`autowiring`的源码分析，我们可以看出，`autowiring`的实现过程：

**a.**  对Bean的属性迭代调用getBean方法，完成依赖Bean的初始化和依赖注入。

**b.** 将依赖Bean的属性引用设置到被依赖的Bean属性上。

**c.** 将依赖Bean的名称和被依赖Bean的名称存储在IoC容器的集合中。

`Spring IoC`容器的`autowiring`属性自动依赖注入是一个很方便的特性，可以简化开发时的配置，但是凡是都有两面性，自动属性依赖注入也有不足，首先，Bean的依赖关系在配置文件中无法很清楚地看出来，对于维护造成一定困难。其次，由于自动依赖注入是Spring容器自动执行的，容器是不会智能判断的，如果配置不当，将会带来无法预料的后果，所以自动依赖注入特性在使用时还是综合考虑。

## 总结

太长了，总共13200字，代码还是截图，但是干货满满，第一遍没看懂，那就再看一遍，直到看到为止，大家真正的掌握才是王道。