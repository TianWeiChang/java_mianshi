# Spring Boot 事件监听的 4 种实现方式

Spring Boot进行事件监听有四种方式：

1. 手工向`ApplicationContext`中添加监听器
2. 将监听器装载入spring容器
3. 在`application.properties`中配置监听器
4. 通过`@EventListener`注解实现事件监听

讲到事件监听，这里我们说下自定义事件和自定义监听器类的实现方式：

- 自定义事件：继承自`ApplicationEvent`抽象类，然后定义自己的构造器
- 自定义监听：实现`ApplicationListener<T>`接口，然后实现`onApplicationEvent`方法

下面讲下4种事件监听的具体实现:

## 方式1

首先创建MyListener1类

```
public class MyListener1 implements ApplicationListener<MyEvent>
{
 Logger logger = Logger.getLogger(MyListener1.class);
 
 public void onApplicationEvent(MyEvent event)
 {
  logger.info(String.format("%s监听到事件源：%s.", MyListener1.class.getName(), event.getSource()));
 }
}
```

然后在springboot应用启动类中获取ConfigurableApplicationContext上下文，装载监听

```
@SpringBootApplication
public class LisenterApplication
{
 public static void main(String[] args)
 {
  ConfigurableApplicationContext context = SpringApplication.run(LisenterApplication.class, args);
  //装载监听
  context.addApplicationListener(new MyListener1());
 }
}
```

> “
>
> 推荐下自己做的 Spring Boot 的实战项目：
>
> https://github.com/YunaiV/ruoyi-vue-pro

## 方式2

创建MyListener2类，并使用@Component注解将该类装载入spring容器中

```
@Component
public class MyListener2 implements ApplicationListener<MyEvent>
{
 Logger logger = Logger.getLogger(MyListener2.class);
 
 public void onApplicationEvent(MyEvent event)
 {
  logger.info(String.format("%s监听到事件源：%s.", MyListener2.class.getName(), event.getSource()));
 }
}
```

> “
>
> 推荐下自己做的 Spring Cloud 的实战项目：
>
> https://github.com/YunaiV/onemall

## 方式3

首先创建MyListener3类

```
public class MyListener3 implements ApplicationListener<MyEvent>
{
 Logger logger = Logger.getLogger(MyListener3.class);
 
 public void onApplicationEvent(MyEvent event)
 {
  logger.info(String.format("%s监听到事件源：%s.", MyListener3.class.getName(), event.getSource()));
 }
}
```

然后在`application.properties`中配置监听

```
context.listener.classes=com.listener.MyListener3
```

## 方式4

创建MyListener4类，该类无需实现ApplicationListener接口，使用@EventListener装饰具体方法

```
@Component
public class MyListener4
{
 Logger logger = Logger.getLogger(MyListener4.class);
 
 @EventListener
 public void listener(MyEvent event)
 {
  logger.info(String.format("%s监听到事件源：%s.", MyListener4.class.getName(), event.getSource()));
 }
}
```

自定义事件代码如下：

```
@SuppressWarnings("serial")
public class MyEvent extends ApplicationEvent
{
 public MyEvent(Object source)
 {
  super(source);
 }
}
```

进行测试(在启动类中加入发布事件的逻辑)：

```
@SpringBootApplication
public class LisenterApplication
{
 public static void main(String[] args)
 {
  ConfigurableApplicationContext context = SpringApplication.run(LisenterApplication.class, args);
  //装载事件
  context.addApplicationListener(new MyListener1());
  //发布事件
  context.publishEvent(new MyEvent("测试事件."));
 }
}
```

启动后，日志打印如下：

```
2018-06-15 10:51:20.198  INFO 4628 --- [           main] com.listener.MyListener3                 : com.listener.MyListener3监听到事件源：测试事件..
2018-06-15 10:51:20.198  INFO 4628 --- [           main] com.listener.MyListener4                 : com.listener.MyListener4监听到事件源：测试事件..
2018-06-15 10:51:20.199  INFO 4628 --- [           main] com.listener.MyListener2                 : com.listener.MyListener2监听到事件源：测试事件..
2018-06-15 10:51:20.199  INFO 4628 --- [           main] com.listener.MyListener1                 : com.listener.MyListener1监听到事件源：测试事件..
```

由日志打印可以看出，SpringBoot四种事件的实现方式监听是有序的

完整的代码路径：

> https://github.com/ingorewho/springboot-develope/tree/master/springboot-listener

萨达