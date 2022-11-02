## 面试官：Spring事务失效的场景有哪些？如何解决？

实际项目开发中，如果涉及到多张表操作时，为了保证业务数据的一致性，大家一般都会采用事务机制；好多小伙伴可能只是简单了解一下，遇到事务失效的情况，便会无从下手，溪源此篇文章给大家整理了一下常见Spring事务失效的场景，希望开发过程尽量避免踩坑，造成时间精力的浪费。

溪源按照最基本的使用方式以及常见失效场景优先级整理,先简单介绍一下具体失效场景：

- 注解@Transactional配置的方法非public权限修饰；
- 注解@Transactional所在类非Spring容器管理的bean；
- 注解@Transactional所在类中，注解修饰的方法被类内部方法调用；
- 业务代码抛出异常类型非RuntimeException，事务失效；
- 业务代码中存在异常时，使用try…catch…语句块捕获，而catch语句块没有throw new RuntimeExecption异常;（最难被排查到问题且容易忽略）
- 注解@Transactional中Propagation属性值设置错误即Propagation.NOT_SUPPORTED（一般不会设置此种传播机制）
- mysql关系型数据库，且存储引擎是MyISAM而非InnoDB，则事务会不起作用(基本开发中不会遇到)；

下面基于以上场景，溪源给小伙伴们详细解释；

### 非public权限修饰

参考Spring官方文档介绍，摘要、译文如下：

> When using proxies, you should apply the @Transactional annotation only to methods with public visibility. If you do annotate protected, private or package-visible methods with the @Transactional annotation, no error is raised, but the annotated method does not exhibit the configured transactional settings. Consider the use of AspectJ (see below) if you need to annotate non-public methods.

**译文**

> 使用代理时，您应该只将@Transactional注释应用于具有公共可见性的方法。如果使用@Transactional注释对受保护的、私有的或包可见的方法进行注释，则不会引发错误，但带注释的方法不会显示配置的事务设置。如果需要注释非公共方法，请考虑使用AspectJ（见下文）。

简言之：**@Transactional 只能用于 public 的方法上，否则事务不会失效，如果要用在非 public 方法上，可以开启 AspectJ 代理模式。**

目前，如果@Transactional注解作用在非public方法上，编译器也会给与明显的提示，如图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdqicNulITj5icgia4l2SyicqHEFhFeic4tjfz0splQiaV7EZtjGAbicibMWGicUVA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 非Spring容器管理的bean

基于这种失效场景，有工作经验的大佬基本上是不会存在这种错误的；@Service 注解注释，StudentServiceImpl 类则不会被Spring容器管理，因此即使方法被@Transactional注解修饰，事务也亦然不会生效。

简单举例如下：

```java
//@Service
public class StudentServiceImpl implements StudentService {

    @Autowired
    private StudentMapper studentMapper;

    @Autowired
    private ClassService classService;

    @Override
    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void insertClassByException(StudentDo studentDo) throws CustomException {
        studentMapper.insertStudent(studentDo);
        throw new CustomException();
    }
}
```

### 注解修饰的方法被类内部方法调用

这种失效场景是我们日常开发中最常踩坑的地方；在类A里面有方法a 和方法b， 然后方法b上面用 @Transactional加了方法级别的事务，在方法a里面 调用了方法b， 方法b里面的事务不会生效。为什么会失效呢？：

其实原因很简单，Spring在扫描Bean的时候会自动为标注了@Transactional注解的类生成一个代理类（proxy）,当有注解的方法被调用的时候，实际上是代理类调用的，代理类在调用之前会开启事务，执行事务的操作，但是同类中的方法互相调用，相当于this.B()，此时的B方法并非是代理类调用，而是直接通过原有的Bean直接调用，所以注解会失效。

```java
@Service
public class ClassServiceImpl implements ClassService {

    @Autowired
    private ClassMapper classMapper;

    public void insertClass(ClassDo classDo) throws CustomException {
        insertClassByException(classDo);
    }

    @Override
    @Transactional(propagation = Propagation.REQUIRED)
    public void insertClassByException(ClassDo classDo) throws CustomException {
        classMapper.insertClass(classDo);
        throw new RuntimeException();
    }
}

//测试用例：
@Test
    public void insertInnerExceptionTest() throws CustomException {
       classDo.setClassId(2);
       classDo.setClassName("java_2");
       classDo.setClassNo("java_2");

       classService.insertClass(classDo);
    }
```

测试结果：

```
java.lang.RuntimeException
 at com.qxy.common.service.impl.ClassServiceImpl.insertClassByException(ClassServiceImpl.java:34)
 at com.qxy.common.service.impl.ClassServiceImpl.insertClass(ClassServiceImpl.java:27)
 at com.qxy.common.service.impl.ClassServiceImpl$$FastClassBySpringCGLIB$$a1c03d8.invoke(<generated>) 
```

虽然业务代码报错了，但是数据库中已经成功插入数据，事务并未生效；

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdqu0smbQCCUoQUzXw4Ifxiak8bdX7vKCFvQO8pcwU3NyQqYX0UWiaORdgw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**解决方案**

类内部使用其代理类调用事务方法：以上方法略作改动

```java
public void insertClass(ClassDo classDo) throws CustomException {
//         insertClassByException(classDo);
        ((ClassServiceImpl)AopContext.currentProxy()).insertClassByException(classDo);
    }
//测试用例：
 @Test
    public void insertInnerExceptionTest() throws CustomException {
       classDo.setClassId(3);
       classDo.setClassName("java_3");
       classDo.setClassNo("java_3");

       classService.insertClass(classDo);
    }
```

业务代码抛出异常，数据库未插入新数据，达到我们的目的，成功解决一个事务失效问题；

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdqNIajbnUujMFvIeXjjGfiakbEUo2lJnEUMGloPoicQZTibFWA7CwdVKQ0w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

数据库数据未发生改变；

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdq3TVCd4xXK1EOu9rs7siauQNlHAjao9HUJnIMxb2Tvo7LtmYnWWnCRTA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

注意：一定要注意启动类上要添加`@EnableAspectJAutoProxy(exposeProxy = true)`注解，否则启动报错：

```
java.lang.IllegalStateException: Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available.

 at org.springframework.aop.framework.AopContext.currentProxy(AopContext.java:69)
 at com.qxy.common.service.impl.ClassServiceImpl.insertClass(ClassServiceImpl.java:28)
```

### 异常类型非RuntimeException

这种事务失效场景也是非常难排查问题的，如果没有深究源码实现，估计要花费一番功夫啦；

```java
@Service
public class ClassServiceImpl implements ClassService {

    @Autowired
    private ClassMapper classMapper;

//    @Override
//    @Transactional(propagation = Propagation.NESTED, rollbackFor = Exception.class)
    public void insertClass(ClassDo classDo) throws Exception {
//        即使此处使用代理对象调用内部事务方法，数据依然未发生回滚，事务机制亦然失效
        ((ClassServiceImpl)AopContext.currentProxy()).insertClassByException(classDo);
    }

    @Override
    @Transactional(propagation = Propagation.REQUIRED)
    public void insertClassByException(ClassDo classDo) throws Exception {
        classMapper.insertClass(classDo);
        //抛出非RuntimeException类型
        throw new Exception();
    }
//测试用例：
 @Test
    public void insertInnerExceptionTest() throws Exception {
       classDo.setClassId(3);
       classDo.setClassName("java_3");
       classDo.setClassNo("java_3");

       classService.insertClass(classDo);
    }
}
```

运行结果：

业务代码抛出异常，但是数据库发生更新操作；

```
java.lang.Exception
 at com.qxy.common.service.impl.ClassServiceImpl.insertClassByException(ClassServiceImpl.java:35)
 at com.qxy.common.service.impl.ClassServiceImpl$$FastClassBySpringCGLIB$$a1c03d8.invoke(<generated>)
 at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
```

数据库依然插入数据，不是我们想要的结果啊，赶紧修改吧，产品经理来追啦~

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdqZRAyX9b9Aw93enT6jQKn2vR4Ppzg7lknNwxbXNh5ZPAp5Fibib6J4e9Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**解决方案：**

@Transactional注解修饰的方法，加上rollbackfor属性值，指定回滚异常类型：`@Transactional(propagation = Propagation.REQUIRED,rollbackFor = Exception.class)`

```java
@Override
@Transactional(propagation = Propagation.REQUIRED,rollbackFor = Exception.class)
public void insertClassByException(ClassDo classDo) throws Exception {
    classMapper.insertClass(classDo);
    throw new Exception();
}
```

### 捕获异常后，却未抛出异常

在事务方法中使用try-catch，导致异常无法抛出，自然会导致事务失效。

```java
@Service
public class ClassServiceImpl implements ClassService {

    @Autowired
    private ClassMapper classMapper;

//    @Override
    public void insertClass(ClassDo classDo) {
        ((ClassServiceImpl)AopContext.currentProxy()).insertClassByException(classDo);

    }

    @Override
    @Transactional(propagation = Propagation.REQUIRED,rollbackFor = Exception.class)
    public void insertClassByException(ClassDo classDo) {
        classMapper.insertClass(classDo);
        try {
            int i = 1 / 0;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

// 测试用例：
 @Test
    public void insertInnerExceptionTest() {
       classDo.setClassId(4);
       classDo.setClassName("java_4");
       classDo.setClassNo("java_4");

       classService.insertClass(classDo);
    }
```

执行结果：

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdqlUrhHR5J6E3nFibj07fsWiaR5xCZUASxOUsYwpUORZ0VdEDVwsNzOrcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAhxRWNvPncTkFjg4q1ibHdqUialfmx5ZQBibGD6xkXtKtplicDFZajjziaRoiciaIXdlyvzQW5gv1zQ5TIQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

解决方案：捕获异常并抛出异常

```java
@Override
@Transactional(propagation = Propagation.REQUIRED,rollbackFor = Exception.class)
public void insertClassByException(ClassDo classDo) {
    classMapper.insertClass(classDo);
    try {
        int i = 1 / 0;
    } catch (Exception e) {
        e.printStackTrace();
        throw new RuntimeException();
    }
}
```

### 事务传播行为设置异常

此种事务传播行为不是特殊自定义设置，基本上不会使用Propagation.NOT_SUPPORTED，不支持事务

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED,rollbackFor = Exception.class)
public void insertClassByException(ClassDo classDo) {
    classMapper.insertClass(classDo);
    try {
        int i = 1 / 0;
    } catch (Exception e) {
        e.printStackTrace();
        throw new RuntimeException();
    }
}
```

### 数据库存储引擎不支持事务

以MySQL关系型数据为例，如果其存储引擎设置为 MyISAM，则事务失效，因为MyISMA 引擎是不支持事务操作的；

故若要事务生效，则需要设置存储引擎为InnoDB ；目前 MySQL 从5.5.5版本开始默认存储引擎是：InnoDB；

来源：blog.csdn.net/xuan_lu/article/details/107797505