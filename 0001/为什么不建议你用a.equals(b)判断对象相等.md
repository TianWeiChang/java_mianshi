为什么不建议你用a.equals(b)判断对象相等

## Objects.equals(a,b)的说明 

一直以为这个方法是java8的，今天才知道是是1.7的时候，然后翻了一下源码。

全路径名为：`java.util.Objects`

![1621579406054](E:\other\网络\assets\1621579406054.png)

这片文章中会总结一下与a.equals(b)的区别，然后对源码做一个小分析。

## 值是null的情况

三种场景：

- a.equals(b), a 是null, 抛出NullPointException异常。
- a.equals(b), a不是null, b是null,  返回false
- Objects.equals(a, b)比较时， 若a 和 b 都是null, 则返回 true, 如果a 和 b 其中一个是null, 另一个不是null, 则返回false。注意：不会抛出空指针异常。

### 使用

```java
null.equals("abc")    →   抛出 NullPointerException 异常
"abc".equals(null)    →   返回 false
null.equals(null)     →   抛出 NullPointerException 异常

Objects.equals(null, "abc")    →   返回 false
Objects.equals("abc",null)     →   返回 false
Objects.equals(null, null)     →   返回 true
```



## 值是空字符串的情况

1.a 和 b 如果都是空值字符串："", 则 `a.equals(b)`, 返回的值是true, 如果a和b其中有一个不是空值字符串，则返回false;

2.这种情况下 `Objects.equals` 与情况1 行为一致。

```java
"abc".equals("")    →   返回 false
"".equals("abc")    →   返回 false
"".equals("")       →   返回 true
    
Objects.equals("abc", "")    →   返回 false
Objects.equals("","abc")     →   返回 false
Objects.equals("","")        →   返回 true
```



## 源码分析

### 源码

```java

public final class Objects {
    private Objects() {
        throw new AssertionError("No java.util.Objects instances for you!");
    }
 
    /**
     * Returns {@code true} if the arguments are equal to each other
     * and {@code false} otherwise.
     * Consequently, if both arguments are {@code null}, {@code true}
     * is returned and if exactly one argument is {@code null}, {@code
     * false} is returned.  Otherwise, equality is determined by using
     * the {@link Object#equals equals} method of the first
     * argument.
     *
     * @param a an object
     * @param b an object to be compared with {@code a} for equality
     * @return {@code true} if the arguments are equal to each other
     * and {@code false} otherwise
     * @see Object#equals(Object)
     */
    public static boolean equals(Object a, Object b) {
        return (a == b) || (a != null && a.equals(b));
    }
    ....省略
}
```

### 源码解释

首先，进行了对象地址的判断，如果是真，则不再继续判断。

如果不相等，后面的表达式的意思是，先判断a不为空，然后根据上面的知识点，就不会再出现空指针。

所以，如果都是null，在第一个判断上就为true了。如果不为空，地址不同，就重要的是判断a.equals(b)。

另外，在Objects中还有一些方法大家也可以用，比如判断是否空等，如果感兴趣可以去看看，源码还是很简单的。

## “a==b”和”a.equals(b)”有什么区别？

如果 a 和 b 都是对象，则 a==b 是比较两个对象的引用，只有当 a 和 b 指向的是堆中的同一个对象才会返回 true。

而 a.equals(b) 是进行逻辑比较，当内容相同时，返回true，所以通常需要重写该方法来提供逻辑一致性的比较。

另外，这里提一下：如果比较的对象没有重写equals方法，那其实就是a.equals(b)和a==b是一样的。

在Object的equals方法中是这样判断的：

```java
public boolean equals(Object obj) {
        return (this == obj);
}
```

再说一个另外：在String类进行了equals方法的重写，并且equals方法中包含了a==b这个判断。

```java
//String中的
public boolean equals(Object anObject) {
    //先判断a==b
    if (this == anObject) {
        return true;
    }
    //在判断类型是否为String类型
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        //再判断长度
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            //在逐个char判断
            while (n-- != 0) {
                //如果存在不一样的，则返回false
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

## 总结

如果需要为null时抛出异常，则使用Object的equals方法进行比较，反之，则建议使用Objects的equals方法。

切记：别搞混了，一个是`java.util.Objects`，一个`java.lang.Object`。











