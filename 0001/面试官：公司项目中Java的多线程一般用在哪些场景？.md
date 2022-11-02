萨达

## 多线程使用场景

多线程使用的主要目的在于：

1、吞吐量：你做WEB，容器帮你做了多线程，但是他只能帮你做请求层面的。简单的说，可能就是一个请求一个线程。或多个请求一个线程。如果是单线程，那同时只能处理一个用户的请求。

2、伸缩性：也就是说，你可以通过增加CPU核数来提升性能。如果是单线程，那程序执行到死也就利用了单核，肯定没办法通过增加CPU核数来提升性能。

鉴于是做WEB的，第1点可能你几乎不涉及。那这里我就讲第二点吧。

### 举个简单的例子：

假设有个请求，这个请求服务端的处理需要执行3个很缓慢的IO操作（比如数据库查询或文件查询），那么正常的顺序可能是（括号里面代表执行时间）：

1. 读取文件1  （10ms）
2. 处理1的数据（1ms）
3. 读取文件2  （10ms）
4. 处理2的数据（1ms）
5. 读取文件3  （10ms）
6. 处理3的数据（1ms）
7. 整合1、2、3的数据结果 （1ms）

单线程总共就需要34ms。

那如果你在这个请求内，把ab、cd、ef分别分给3个线程去做，就只需要12ms了。

所以多线程不是没怎么用，而是，你平常要善于发现一些可优化的点。然后评估方案是否应该使用。假设还是上面那个相同的问题：但是每个步骤的执行时间不一样了。

1. 读取文件1  （1ms）
2. 处理1的数据（1ms）
3. 读取文件2  （1ms）
4. 处理2的数据（1ms）
5. 读取文件3  （28ms）
6. 处理3的数据（1ms）
7. 整合1、2、3的数据结果 （1ms）

单线程总共就需要34ms。

如果还是按上面的划分方案（上面方案和木桶原理一样，耗时取决于最慢的那个线程的执行速度），在这个例子中是第三个线程，执行29ms。那么最后这个请求耗时是30ms。比起不用单线程，就节省了4ms。但是有可能线程调度切换也要花费个1、2ms。因此，这个方案显得优势就不明显了，还带来程序复杂度提升。不太值得。

那么现在优化的点，就不是第一个例子那样的任务分割多线程完成。而是优化文件3的读取速度。可能是采用缓存和减少一些重复读取。

首先，假设有一种情况，所有用户都请求这个请求，那其实相当于所有用户都需要读取文件3。那你想想，100个人进行了这个请求，相当于你花在读取这个文件上的时间就是28×100=2800ms了。那么，如果你把文件缓存起来，那只要第一个用户的请求读取了，第二个用户不需要读取了，从内存取是很快速的，可能1ms都不到。

伪代码：

```java
public class MyServlet extends Servlet{
    private static Map<String, String> fileName2Data = new HashMap<String, String>();
    private void processFile3(String fName){
        String data = fileName2Data.get(fName);
        if(data==null){
            data = readFromFile(fName);    //耗时28ms
            fileName2Data.put(fName, data);
        }
        //process with data
    }
}
```

看起来好像还不错，建立一个文件名和文件数据的映射。如果读取一个map中已经存在的数据，那么就不不用读取文件了。

可是问题在于，Servlet是并发，上面会导致一个很严重的问题，死循环。因为，HashMap在并发修改的时候，可能是导致循环链表的构成！！！（具体你可以自行阅读HashMap源码）如果你没接触过多线程，可能到时候发现服务器没请求也巨卡，也不知道什么情况！

好的，那就用ConcurrentHashMap，正如他的名字一样，他是一个线程安全的HashMap，这样能轻松解决问题。

```java
public class MyServlet extends Servlet{
    private static ConcurrentHashMap<String, String> fileName2Data = new ConcurrentHashMap<String, String>();
    private void processFile3(String fName){
        String data = fileName2Data.get(fName);
        if(data==null){
            data = readFromFile(fName);    //耗时28ms
            fileName2Data.put(fName, data);
        }
        //process with data
    }
}
```

这样真的解决问题了吗，这样虽然只要有用户访问过文件a，那另一个用户想访问文件a，也会从fileName2Data中拿数据，然后也不会引起死循环。

可是，如果你觉得这样就已经完了，那你把多线程也想的太简单了，骚年！你会发现，1000个用户首次访问同一个文件的时候，居然读取了1000次文件（这是最极端的，可能只有几百）。What the fuckin hell!!!

难道代码错了吗，难道我就这样过我的一生！

好好分析下。Servlet是多线程的，那么

```java
public class MyServlet extends Servlet{
    private static ConcurrentHashMap<String, String> fileName2Data = new ConcurrentHashMap<String, String>();
    private void processFile3(String fName){
        String data = fileName2Data.get(fName);
        //“偶然”-- 1000个线程同时到这里，同时发现data为null
        if(data==null){
            data = readFromFile(fName);    //耗时28ms
            fileName2Data.put(fName, data);
        }
        //process with data
    }
}
```

上面注释的“偶然”，这是完全有可能的，因此，这样做还是有问题。

因此，可以自己简单的封装一个任务来处理。

```java
public class MyServlet extends Servlet{
    private static ConcurrentHashMap<String, FutureTask> fileName2Data = new ConcurrentHashMap<String, FutureTask>();
    private static ExecutorService exec = Executors.newCacheThreadPool();
    private void processFile3(String fName){
        FutureTask data = fileName2Data.get(fName);
        //“偶然”-- 1000个线程同时到这里，同时发现data为null
        if(data==null){
            data = newFutureTask(fName);
            FutureTask old = fileName2Data.putIfAbsent(fName, data);
            if(old==null){
                data = old;
            }else{
                exec.execute(data);
            }
        }
        String d = data.get();
        //process with data
    }
     
    private FutureTask newFutureTask(final String file){
        return  new FutureTask(new Callable<String>(){
            public String call(){
                return readFromFile(file);
            }
 
            private String readFromFile(String file){return "";}
        }
    }
}
```

以上所有代码都是直接在bbs打出来的，不保证可以直接运行。

多线程最多的场景：web服务器本身；各种专用服务器（如游戏服务器）；

多线程的常见应用场景：

- 后台任务，例如：定时向大量（100w以上）的用户发送邮件；
- 异步处理，例如：发微博、记录日志等；
- 分布式计算



来源：cnblogs.com/kenshinobiy/p/4671314.html