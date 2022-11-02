Spring面试中常见的9种设计模式！

## 一、简单工厂

又叫做静态工厂方法（StaticFactory Method）模式，但不属于23种GOF设计模式之一。

简单工厂模式的实质是由一个工厂类根据传入的参数，动态决定应该创建哪一个产品类。

Spring中的BeanFactory就是简单工厂模式的体现，根据传入一个唯一的标识来获得Bean对象，但是否是在传入参数后创建还是传入参数前创建这个要根据具体情况来定。

## 二、工厂方法（Factory Method）

定义一个用于创建对象的接口，让子类决定实例化哪一个类。Factory Method使一个类的实例化延迟到其子类。

Spring中的FactoryBean就是典型的工厂方法模式。如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWF3GpJqZ0o8qIP9SKzw8CAYS3PyXoNoh5EZ7aobBWribLUUKk32ROBgw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 三、单例（Singleton）

保证一个类仅有一个实例，并提供一个访问它的全局访问点。

Spring中的单例模式完成了后半句话，即提供了全局的访问点BeanFactory。但没有从构造器级别去控制单例，这是因为Spring管理的是是任意的Java对象。

四、适配器（Adapter）

将一个类的接口转换成客户希望的另外一个接口。Adapter模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

Spring中在对于AOP的处理中有Adapter模式的例子，见如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWz6cKg7yI52yacK6k0sLJubnCMo8Fdudzibkp7icRcjhIARObSS7s4rYA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由于Advisor链需要的是MethodInterceptor（拦截器）对象，所以每一个Advisor中的Advice都要适配成对应的MethodInterceptor对象。

## 五、包装器（Decorator）

动态地给一个对象添加一些额外的职责。就增加功能来说，Decorator模式相比生成子类更为灵活。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWBlQfLOiaLLZepc4FTbb4brFbO36vcHONHKCJU4PhMTwsmQfaIibaiajqQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Spring中用到的包装器模式在类名上有两种表现：一种是类名中含有Wrapper，另一种是类名中含有Decorator。基本上都是动态地给一个对象添加一些额外的职责。

## 六、代理（Proxy）

为其他对象提供一种代理以控制对这个对象的访问。

从结构上来看和Decorator模式类似，但Proxy是控制，更像是一种对功能的限制，而Decorator是增加职责。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWz01sIyHGJc5yn4Xdro6R6kJfdfql4Z5KQpcSdjtd0L36H62xQENWGA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Spring的Proxy模式在aop中有体现，比如JdkDynamicAopProxy和Cglib2AopProxy。

## 七、观察者（Observer）

定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWqyRmCvFoZj3CGlFuypCfjcLpdj4SU9R8ztovUxsqMIl0ibmhJylgiaLA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Spring中Observer模式常用的地方是listener的实现。如ApplicationListener。

## 八、策略（Strategy）

定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。

Spring中在实例化对象的时候用到Strategy模式，见如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWMw4FiaYeGh32tTY6UouNNfLtlPLjDE84d2EyuiaIECfyggOBkVor5APw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在SimpleInstantiationStrategy中有如下代码说明了策略模式的使用情况：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMW9xsNv1q3pk7QK34MLFspNic8nwWnicDxxj5G178w2MwaUSPtkyEibSjQw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 九、模板方法（Template Method）

定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。Template Method使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

Template Method模式一般是需要继承的。这里想要探讨另一种对Template Method的理解。Spring中的JdbcTemplate，在用这个类时并不想去继承这个类，因为这个类的方法太多，但是我们还是想用到JdbcTemplate已有的稳定的、公用的数据库连接，那么我们怎么办呢？我们可以把变化的东西抽出来作为一个参数传入JdbcTemplate的方法中。但是变化的东西是一段代码，而且这段代码会用到JdbcTemplate中的变量。怎么办？那我们就用回调对象吧。在这个回调对象中定义一个操纵JdbcTemplate中变量的方法，我们去实现这个方法，就把变化的东西集中到这里了。然后我们再传入这个回调对象到JdbcTemplate，从而完成了调用。这可能是Template Method不需要继承的另一种实现方式吧。

以下是一个具体的例子：

JdbcTemplate中的execute方法：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWzzl4VRfV8UUGc84G40mJUGVDCPtbELWS6NDLd9H3cIZWot6kpajiaKw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

JdbcTemplate执行execute方法：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHyEkn7O0Zw7Ac0f6yApTMWX7gia2eeQfwP8RjECABNSeldFoib8AROZehAAdmSsWUTNS6TtHM7B5Lg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 后记

以上9种设计模式，是作为一个开发者必备的设计模式，如果这都回答不上来，那你得继续学习了。