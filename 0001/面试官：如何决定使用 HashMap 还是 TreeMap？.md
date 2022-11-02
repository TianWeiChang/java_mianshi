面试官：如何决定使用 HashMap 还是 TreeMap？

`HashMap`是我们开发人员使用的最多的Java集合之一，面试官也特别好这口，一个`HashMap`不知道多少被问倒。

面试场景（虚拟）

> 面试官：你为什么要用`HashMap`，而不用其他Map子类？
>
> 菜鸟：这个没想过，主要是用他的`key-value`存储数据。
>
> 面试官：其他Map子类不能吗？
>
> 菜鸟：额，(⊙o⊙)…

下面我们来分析一下，这道题到底坑在哪里。

### 知识铺垫

Java为数据结构中的映射定义了一个接口`java.util.Map`，此接口主要有四个常用的实现类，分别是`HashMap`、`Hashtable`、`LinkedHashMap`和`TreeMap`，类继承关系如下图所示： 

![img](https://pic2.zhimg.com/80/26341ef9fe5caf66ba0b7c40bba264a5_720w.png) 

## 实现类介绍

`TreeMap`：`TreeMap`实现`SortedMap`接口，能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历`TreeMap`时，得到的记录是排过序的。如果使用排序的映射，建议使用`TreeMap`。在使用`TreeMap`时，key必须实现`Comparable`接口或者在构造`TreeMap`传入自定义的`Comparator`，否则会在运行时抛出`java.lang.ClassCastException`类型的异常。 

`HashMap`：它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 `HashMap`最多只允许一条记录的键为null，允许多条记录的值为null。`HashMap`非线程安全，即任一时刻可以有多个线程同时写`HashMap`，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的`synchronizedMap`方法使`HashMap`具有线程安全的能力，或者使用`ConcurrentHashMap`。 

`Hashtable`：`Hashtable`是遗留类，很多映射的常用功能与`HashMap`类似，不同的是它承自`Dictionary`类，并且是线程安全的，任一时间只有一个线程能写`Hashtable`，并发性不如`ConcurrentHashMap`，因为ConcurrentHashMap引入了分段锁。`Hashtable`不建议在新代码中使用，不需要线程安全的场合可以用`HashMap`替换，需要线程安全的场合可以用`ConcurrentHashMap`替换。 

`LinkedHashMap`：`LinkedHashMap`是`HashMap`的一个子类，保存了记录的插入顺序，在用`Iterator`遍历`LinkedHashMap`时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。 

`AbstractMap`：覆盖了equals()和hashCode()方法以确保两个相等映射返回相同的哈希码。如果两个映射大小相等、包含同样的键且每个键在这两个映射中对应的值都相同，则这两个映射相等。映射的哈希码是映射元素哈希码的总和，其中每个元素是Map.Entry接口的一个实现。因此，不论映射内部顺序如何，两个相等映射会报告相同的哈希码。

`SortedMap`：它用来保持键的有序顺序。`SortedMap`接口为映像的视图(子集)，包括两个端点提供了访问方法。除了排序是作用于映射的键以外，处理`SortedMap`和处理SortedSet一样。添加到`SortedMap`实现类的元素必须实现Comparable接口，否则您必须给它的构造函数提供一个Comparator接口的实现。TreeMap类是它的唯一一个实现。

## 如何选择呢？

回到最初的问题中，为什么使用HashMap，而不使用TreeMap?

**1、HashMap 和 TreeMap 的实现**

**HashMap：**基于哈希表实现。使用`HashMap`要求添加的键类明确定义了`hashCode()`和`equals()`[可以重写`hashCode()`和`equals()`]，为了优化`HashMap`空间的使用，您可以调优初始容量和负载因子。

```java
//构建一个拥有特定容量和加载因子的空的哈希映像
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
}

//构建一个拥有特定容量的空的哈希映像
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//构建一个空的哈希映像 
public HashMap() {
    // all other fields defaulted
    this.loadFactor = DEFAULT_LOAD_FACTOR; 
}

//构建一个哈希映像，并且添加映像m的所有映射 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

**TreeMap：**基于红黑树实现。`TreeMap`没有调优选项，因为该树总处于平衡状态。

```java
//构建一个空的映像树
public TreeMap() {
    comparator = null;
}

// 构建一个映像树，并且使用特定的比较器对关键字进行排序
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}

//构建一个映像树，并且添加映像m中所有元素
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}

//构建一个映像树，添加映像树s中所有映射，并且使用与有序映像s相同的比较器排序
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```

**2、TreeMap如何实现降序**

首先我们需要搞清楚，它是如何做到升序的。

定义一个比较器类，实现`Comparator`接口，重写compare方法，有两个参数，这两个参数通过调用compareTo进行比较，而`compareTo`默认规则是：

> - 如果参数字符串等于此字符串，则返回 0 值；
> - 如果此字符串小于字符串参数，则返回一个小于 0 的值；
> - 如果此字符串大于字符串参数，则返回一个大于 0 的值。

自定义比较器时，在返回时多添加了个负号，就将比较的结果以相反的形式返回，代码如下：

```java
static class MyComparator implements Comparator{
    @Override
    public int compare(Object o1, Object o2) {
        String param1 = (String)o1;
        String param2 = (String)o2;
        return -param1.compareTo(param2);
    }   
}
```

之后，通过`MyComparato`r类初始化一个比较器实例，将其作为参数传进`TreeMap`的构造方法中：

```java
MyComparator comparator = new MyComparator();
Map<String,String> map = new TreeMap<String,String>(comparator);
```

这样，我们就可以使用自定义的比较器实现降序了

```java
public class MapTest {

    public static void main(String[] args) {
        //初始化自定义比较器
        MyComparator comparator = new MyComparator();
        //初始化一个map集合
        Map<String,String> map = new TreeMap<String,String>(comparator);
        //存入数据
        map.put("a", "a");
        map.put("b", "b");
        map.put("f", "f");
        map.put("d", "d");
        map.put("c", "c");
        map.put("g", "g");
        //遍历输出
        Iterator iterator = map.keySet().iterator();
        while(iterator.hasNext()){
            String key = (String)iterator.next();
            System.out.println(map.get(key));
        }
    }

    static class MyComparator implements Comparator{
        @Override
        public int compare(Object o1, Object o2) {
            String param1 = (String)o1;
            String param2 = (String)o2;
            return -param1.compareTo(param2);
        }
    }
}
```

这样，我们就实现了`TreeMap`的降序，其实在工作中，关于比较器使用的也还是有很多场景。

比如：校验客户端穿过来的参数是否被篡改过。

客户端只要把相应参数名和参数值，按照升序排好，再进行`MD5`得出一个字符串。

服务端收到客户端的参数后，也进行排序，再进行`MD5。`

最后比较两个`MD5`的字符串是否一致。

> 通常还要在参数排序好了后，加一个`特殊字符串`，拼接在一起，再进行`MD5`，这样更安全。
>
> 注意`特殊字符串`建议使用邮件形式发给对方。

## 结论

如果需要保证线程安全，建议使用`ConcurrentHashMap`。

如果你需要得到一个有序的结果时就应该使用`TreeMap`。

如果不需要保证线程安全、也不需要有序，请使用`HashMap`，因为`HashMap`性能更好。



好了，今天就分享到这里了



