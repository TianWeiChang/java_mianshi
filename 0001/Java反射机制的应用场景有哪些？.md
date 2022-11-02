## Java反射机制的应用场景有哪些？

个人理解，反射机制实际上就是上帝模式，如果说方法的调用是 Java 正确的打开方式，那反射机制就是上帝偷偷开的后门，只要存在对应的class，一切都能够被调用。

那上帝为什么要打开这个后门呢？这涉及到了静态和动态的概念

- 静态编译：在编译时确定类型，绑定对象
- 动态编译：运行时确定类型，绑定对象

两者的区别在于，动态编译可以最大程度地支持多态，而多态最大的意义在于降低类的耦合性，因此反射的优点就很明显了：解耦以及提高代码的灵活性。

因此，反射的优势和劣势分别在于：

- **优势**
- 运行期类型的判断，动态类加载：提高代码灵活度
- **劣势**
- 性能瓶颈：反射相当于一系列解释操作，通知 JVM 要做的事情，性能比直接的java代码要慢很多

## 反射的应用场景

在我们平时的项目开发过程中，基本上很少会直接使用到反射机制，但这不能说明反射机制没有用，实际上有很多设计、开发都与反射机制有关，例如模块化的开发，通过反射去调用对应的字节码；动态代理设计模式也采用了反射机制，还有我们日常使用的 Spring／Hibernate 等框架，也是利用CGLIB 反射机制才得以实现，下面就举例最常见的两个例子，来说明反射机制的强大之处：

### JDBC 的数据库的连接

在JDBC 的操作中，如果要想进行数据库的连接，则必须按照以上的几步完成

1. 通过Class.forName()加载数据库的驱动程序 （通过反射加载，前提是引入相关了Jar包）
2. 通过 DriverManager 类进行数据库的连接，连接的时候要输入数据库的连接地址、用户名、密码
3. 通过Connection 接口接收连接

```
public class ConnectionJDBC {  
  
    /** 
     * @param args 
     */  
    //驱动程序就是之前在classpath中配置的JDBC的驱动程序的JAR 包中  
    public static final String DBDRIVER = "com.mysql.jdbc.Driver";  
    //连接地址是由各个数据库生产商单独提供的，所以需要单独记住  
    public static final String DBURL = "jdbc:mysql://localhost:3306/test";  
    //连接数据库的用户名  
    public static final String DBUSER = "root";  
    //连接数据库的密码  
    public static final String DBPASS = "";  
      
      
    public static void main(String[] args) throws Exception {  
        Connection con = null; //表示数据库的连接对象  
        Class.forName(DBDRIVER); //1、使用CLASS 类加载驱动程序 ,反射机制的体现 
        con = DriverManager.getConnection(DBURL,DBUSER,DBPASS); //2、连接数据库  
        System.out.println(con);  
        con.close(); // 3、关闭数据库  
    }  
```

###  Spring 框架的使用

在 Java的反射机制在做基础框架的时候非常有用，行内有一句这样的老话：反射机制是Java框架的基石。一般应用层面很少用，不过这种东西，现在很多开源框架基本都已经封装好了，自己基本用不着写。

典型的除了hibernate之外，还有spring也用到很多反射机制。最经典的就是xml的配置模式。

Spring 通过 XML 配置模式装载 Bean 的过程：

1. 将程序内所有 XML 或 Properties 配置文件加载入内存中
2. Java类里面解析xml或properties里面的内容，得到对应实体类的字节码字符串以及相关的属性信息
3. 使用反射机制，根据这个字符串获得某个类的Class实例
4. 动态配置实例的属性

Spring这样做的好处是：

- 不用每一次都要在代码里面去new或者做其他的事情
- 以后要改的话直接改配置文件，代码维护起来就很方便了
- 有时为了适应某些需求，Java类里面不一定能直接调用另外的方法，可以通过反射机制来实现

模拟 Spring 加载 XML 配置文件：

```java
public class BeanFactory {
       private Map<String, Object> beanMap = new HashMap<String, Object>();
       /**
       * bean工厂的初始化.
       * @param xml xml配置文件
       */
       public void init(String xml) {
              try {
                     //读取指定的配置文件
                     SAXReader reader = new SAXReader();
                     ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
                     //从class目录下获取指定的xml文件
                     InputStream ins = classLoader.getResourceAsStream(xml);
                     Document doc = reader.read(ins);
                     Element root = doc.getRootElement();  
                     Element foo;
                    
                     //遍历bean
                     for (Iterator i = root.elementIterator("bean"); i.hasNext();) {  
                            foo = (Element) i.next();
                            //获取bean的属性id和class
                            Attribute id = foo.attribute("id");  
                            Attribute cls = foo.attribute("class");
                           
                            //利用Java反射机制，通过class的名称获取Class对象
                            Class bean = Class.forName(cls.getText());
                           
                            //获取对应class的信息
                            java.beans.BeanInfo info = java.beans.Introspector.getBeanInfo(bean);
                            //获取其属性描述
                            java.beans.PropertyDescriptor pd[] = info.getPropertyDescriptors();
                            //设置值的方法
                            Method mSet = null;
                            //创建一个对象
                            Object obj = bean.newInstance();
                           
                            //遍历该bean的property属性
                            for (Iterator ite = foo.elementIterator("property"); ite.hasNext();) {  
                                   Element foo2 = (Element) ite.next();
                                   //获取该property的name属性
                                   Attribute name = foo2.attribute("name");
                                   String value = null;
                                  
                                   //获取该property的子元素value的值
                                   for(Iterator ite1 = foo2.elementIterator("value"); ite1.hasNext();) {
                                          Element node = (Element) ite1.next();
                                          value = node.getText();
                                          break;
                                   }
                                  
                                   for (int k = 0; k < pd.length; k++) {
                                          if (pd[k].getName().equalsIgnoreCase(name.getText())) {
                                                 mSet = pd[k].getWriteMethod();
                                                 //利用Java的反射极致调用对象的某个set方法，并将值设置进去
                                                 mSet.invoke(obj, value);
                                          }
                                   }
                            }
                           
                            //将对象放入beanMap中，其中key为id值，value为对象
                            beanMap.put(id.getText(), obj);
                     }
              } catch (Exception e) {
                     System.out.println(e.toString());
              }
       }
      
       //other codes
}
```

其实，对于反射机制的使用，在很多框架中都用用到，Dubbo、Mybatis以及文章中提到的Spring。