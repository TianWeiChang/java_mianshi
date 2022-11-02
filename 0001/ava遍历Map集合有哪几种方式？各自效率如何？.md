## 1、由来

我们应该在什么时刻选择什么样的遍历方式呢，必须通过实践的比较才能看到效率，也看了很多文章，大家建议使用entrySet，认为entrySet对于大数据量的查找来说，速度更快，今天我们就通过下面采用不同方法遍历key+value，key，value不同情景下的差异。

## 2、准备测试数据：

HashMap1：大小为1000000，key和value的值均为String，key的值为1、2、3.........1000000；

```java
Map<String,String> map =new HashMap<String,String>();
String key,value;

for(int i=1;i<=num;i++){
    key = ""+i;
    value="value"+i;
   map.put(key,value);
}
```

HashMap2：大小为1000000，key和value的值为String，key的值为50、100、150........50000000；

```java
Map<String,String> map = new HashMap<String,String>();
String key,value;

for(int i=1;i<=num;i++){
   key=""+(i*50);
   value="value"+key;
   map.put(key,value);
}
```

## 3、场景测试

### 3.1遍历key+value

1）keySet利用Iterator遍历

```java
long startTime1 =System.currentTimeMillis();
Iterator<String> iter = map.keySet().iterator();
while (iter.hasNext()){
      key=iter.next();
      value=map.get(key);
}
long endTime1 =System.currentTimeMillis();
System.out.println("第一个程序运行时间："+(endTime1-startTime1)+"ms");
```

2）keySet利用for遍历

```java
long startTime2 =System.currentTimeMillis();
  for(String key2:map.keySet()){
      value=map.get(key2);
  }
long endTime2 =System.currentTimeMillis();
System.out.println("第二个程序运行时间："+(endTime2-startTime2)+"ms");
```

3）entrySet利用Iterator遍历

```java
long startTime3=System.currentTimeMillis();
Iterator<Map.Entry<String,String>> iter3 =map.entrySet().iterator();
Map.Entry<String,String> entry3;
while (iter3.hasNext()){
    entry3 = iter3.next();
    key = entry3.getKey();
    value=entry3.getValue();
}
long endTime3 =System.currentTimeMillis();
System.out.println("第三个程序运行时间：" +(endTime3-startTime3)+"ms");
```

4）entrySet利用for遍历

```java
long startTime4=System.currentTimeMillis();
for(Map.Entry<String,String> entry4:map.entrySet()){
    key=entry4.getKey();
    value=entry4.getValue();
}
long endTime4 =System.currentTimeMillis();
System.out.println("第四个程序运行时间："+(endTime4-startTime4) +"ms");
```

### 3.2遍历key

1）keySet利用Iterator遍历

```java
long startTime1 =System.currentTimeMillis();
Iterator<String> iter = map.keySet().iterator();
while (iter.hasNext()){
    key=iter.next();

}
long endTime1 =System.currentTimeMillis();
System.out.println("第一个程序运行时间："+(endTime1-startTime1)+"ms");
```

2）keySet利用for遍历

```java
long startTime2 =System.currentTimeMillis();
for(String key2:map.keySet()){

}
long endTime2 =System.currentTimeMillis();
System.out.println("第二个程序运行时间："+(endTime2-startTime2)+"ms");
```

3）entrySet利用Iterator遍历

```java
 long startTime3=System.currentTimeMillis();
Iterator<Map.Entry<String,String>> iter3 =map.entrySet().iterator();
Map.Entry<String,String> entry3;
while (iter3.hasNext()){
    key = iter3.next().getKey();

}
long endTime3 =System.currentTimeMillis();
System.out.println("第三个程序运行时间：" +(endTime3-startTime3)+"ms");
```

4）entrySet利用for遍历

```java
 long startTime4=System.currentTimeMillis();
for(Map.Entry<String,String> entry4:map.entrySet()){
    key=entry4.getKey();
}
long endTime4 =System.currentTimeMillis();
System.out.println("第四个程序运行时间："+(endTime4-startTime4) +"ms");
```

### 3.3遍历value

1）keySet利用Iterator遍历

```java
long startTime1 =System.currentTimeMillis();
Iterator<String> iter = map.keySet().iterator();
while (iter.hasNext()){
   value=map.get(iter.next());
}
long endTime1 =System.currentTimeMillis();
System.out.println("第一个程序运行时间："+(endTime1-startTime1)+"ms");
```

2）keySet利用for遍历

```java
 long startTime2 =System.currentTimeMillis();
for(String key2:map.keySet()){
    value=map.get(key2);
}
long endTime2 =System.currentTimeMillis();
System.out.println("第二个程序运行时间："+(endTime2-startTime2)+"ms");
```

3）entrySet利用Iterator遍历

```java
 long startTime3=System.currentTimeMillis();
Iterator<Map.Entry<String,String>> iter3 =map.entrySet().iterator();
Map.Entry<String,String> entry3;
while (iter3.hasNext()){
   value=iter3.next().getValue();

}
long endTime3 =System.currentTimeMillis();
System.out.println("第三个程序运行时间：" +(endTime3-startTime3)+"ms");
```

4）entrySet利用for遍历

```java
long startTime4=System.currentTimeMillis();
for(Map.Entry<String,String> entry4:map.entrySet()){
    value=entry4.getValue();
}
long endTime4 =System.currentTimeMillis();
System.out.println("第四个程序运行时间："+(endTime4-startTime4) +"ms");
```

5）values利用iterator遍历

```java
 long startTime5=System.currentTimeMillis();
Iterator<String>  iter5=map.values().iterator();
while (iter5.hasNext()){
    value=iter5.next();
}
long endTime5 =System.currentTimeMillis();
System.out.println("第五个程序运行时间："+(endTime5-startTime5) +"ms");
```

6）values利用for遍历

```java
long startTime6=System.currentTimeMillis();
for(String value6:map.values()){

}
long endTime6 =System.currentTimeMillis();
System.out.println("第六个程序运行时间："+(endTime6-startTime6) +"ms");
```

## 4、时间对比

### 4.1遍历key+value

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XC01gAmYI0ZcjEiaZevYjnzMneicrFSsZkUfTUHH09ANrRwTc2vObEQ8GfVDlIYiaicybzqlaYakdxKkQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.2遍历key 

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XC01gAmYI0ZcjEiaZevYjnzMagBYeicHczxfLcnYS47jGGhUfoibOlyWzA4c3DEaNPzDKUjPkj8nqUnA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.3遍历value 

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XC01gAmYI0ZcjEiaZevYjnzMNVH0TXXh0NicgkdlNWMKKmqFIyZDCuLQZfjsicepvYcaa2DNW8dib05aw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 5、总结 

从上面的时间比较来看：

1）map的key采用简单形式和复杂形式时，查找的效率是不同的，简单的key值效率更高

2）当数据量大的时候，采用entrySet遍历key+value的效率要高于keySet

3）当我们只需要取得value值时，采用values来遍历效率更高