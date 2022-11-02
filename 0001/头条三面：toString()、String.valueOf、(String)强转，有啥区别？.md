头条三面：toString()、String.valueOf、(String)强转，有啥区别？

## 一、前言

相信大家在日常开发中这三种方法用到的应该很多，尤其是前两种，经常在开发的时候，随心所欲，想用哪个用哪个，既然存在，那就应该有它存在的道理，那么什么情况下用哪个呢？ 

## 二、代码实例

### 1、基本类型

**（1）基本类型没有toString()方法**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFGq77eHYvmZ1JE6Y3Bydky1V4ZheuicSeGcUw0b3qo5WJsRFlYicGOMNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**（2）推荐使用**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFmqJH34VG215iaJgtNxz8AA1ODAG24jql7vED46qFBV5UBlzzMYogicYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**（3）无法强转**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFlQW3YhZtLhoLrY2SiawE4lmQtaVfx23h6qpiaibXiaEJQaZ5EY7lBxZBgA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

（String）是标准的类型转换，将Object类型转为String类型，使用(String)强转时，最好使用instanceof做一个类型检查，以判断是否可以进行强转，否则容易抛出ClassCastException异常。需要注意的是编写的时候，编译器并不会提示有语法错误，所以这个方法要谨慎的使用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFZNS21Csrevzl5GrBicUMZjDWEJiaDFlF291XpxZgrrtYT2dMiaZQtJLpQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

instanceof判断

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGF7qsNehQXV95uQOpGCWlQXtibXIlgfLTZs8Fln1MlvA4vtkWpyTgVjibQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2、封装类型

**（1）toString ok**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFaj3RzJmRrKbOPcrTpcxMq16EC4C5USAjRbnwDvYaMDtwjjf2DtZLUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**（2）String.valueOf()**

自然也是可以的。推荐：Java进阶视频资源

**（3）封装类型也无法强转**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFOmtv9dtN4rbPCo48DK5KnnBzyBmkkT38gEmmE7LzCg8NxWISO6e46Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3、null值问题

**（1）toString()报空指针异常**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFMhkqibnbFqO8pOic6RE536icmEClMrapUyCOpMh4APcSX9txKvJ0gyG2g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**（2）String.valueOf()返回字符串“null”**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFHD9jF3oFNzWibTvTr7Qgsp8CpMcGOUH5eysZqvP9yDyfXYCwdr3LL7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**（3）null值强转成功**

**在公众号Java后端栈后台回复“面试”，获取一份惊喜礼包。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFPHicp4icejRGpS5vE6VL0GiaY78jmOvw0Ag4qibcziba1ic9CRR6SlhliaHrA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 三、源码分析

### 1、toString()

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFQwiaqe720erjIL5VjJXgYjibib21PqkH9Z0jh0QDBjP99zmZTpaPT45Sg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGFflqrkXzRRdNyAYTYCSdQbIwNWibdGO02T4GXaMwetHm5aibrHnibMcBOQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2、String.valueOf()

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XDzNRmFGsCs9xOo0PD6zLGF7oM0nTiapwhptmJ9zYXpdaCGd0B3ibMk1ccVPf7n7JKXgnY8l8lBWLiag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`String.valueOf()`比toString多了一个非空判断。

## 四、总结

### 1、toString()，可能会抛空指针异常

在这种使用方法中，因为`java.lang.Object`类里已有`public方法.toString()`，所以`Java`对象都可以调用此方法。但在使用时要注意，必须保证object不是null值，否则将抛出`NullPointerException`异常。采用这种方法时，通常派生类会覆盖Object里的`toString()`方法。

### 2、String.valueOf()，推荐使用，返回字符串“null”

`String.valueOf()`方法是小编推荐使用的，因为它不会出现空指针异常，而且是静态的方法，直接通过String调用即可，只是有一点需要注意，就是上面提到的，如果为null，`String.valueOf()`返回结果是字符串“null”。而不是null。

### 3、(String)强转，不推荐使用

（String）是标准的类型转换，将Object类型转为String类型，使用(String)强转时，最好使用`instanceof`做一个类型检查，以判断是否可以进行强转，否则容易抛出`ClassCastException`异常。需要注意的是编写的时候，编译器并不会提示有语法错误，所以这个方法要谨慎的使用。





























