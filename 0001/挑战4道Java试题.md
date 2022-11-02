## 挑战4道Java试题

四道Java基础题，你能对几道？

**一、==符的使用**首先看一段比较有意思的代码

```java
Integer a = 1000,b=1000;
Integer c = 100,d=100;  

public void mRun(final String name){
    new Runnable() {
        
      public void run() {
        System.out.println(name);
      }
    };
  }
    
  
System.out.println(a==b);
System.out.println(c==d);
```

