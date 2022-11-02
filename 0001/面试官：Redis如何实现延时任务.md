面试官：Redis如何实现延时任务？

## 什么是延时任务

延时任务，顾名思义，就是延迟一段时间后才执行的任务。举个例子，假设我们有个发布资讯的功能，运营需要在每天早上7点准时发布资讯，但是早上7点大家都还没上班，这个时候就可以使用延时任务来实现资讯的延时发布了。只要在前一天下班前指定第二天要发送资讯的时间，到了第二天指定的时间点资讯就能准时发出去了。如果大家有运营过公众号，就会知道公众号后台也有文章定时发送的功能。总而言之，延时任务的使用还是很广泛的。关于延时任务的实现方式，我知道的就不下于3种，后面会逐一介绍，今天就讲下如何用redis实现延时任务。

## 延时任务的特点

在介绍具体方案之前，我们不妨先想一下要实现一个延时系统，有哪些内容是必须存储下来的（这里的存储不一定是指持久化，也可以是放在内存中，取决于延时任务的重要程度）。首先要存储的就是**任务的描述**。假如你要处理的延时任务是延时发布资讯，那么你至少要存储资讯的id吧。另外，如果你有多种任务类型，比如：延时推送消息、延时清洗数据等等，那么你还需要存储任务的类型。以上总总，都归属于任务描述。除此之外，你还必须存储**任务执行的时间点**吧，一般来说就是时间戳。此外，我们还需要根据任务的执行时间进行排序，因为延时任务队列里的任务可能会有很多，只有到了时间点的任务才应该被执行，所以必须支持对任务执行时间进行排序。

## 使用Redis实现延时任务

以上就是一个延迟任务系统必须具备的要素了。回到redis，有什么数据结构可以既存储任务描述，又能存储任务执行时间，还能根据任务执行时间进行排序呢？想来想去，似乎只有**Sorted Set**了。我们可以把任务的描述序列化成字符串，放在Sorted Set的value中，然后把任务的执行时间戳作为score，利用Sorted Set天然的排序特性，执行时刻越早的会排在越前面。这样一来，我们只要开一个或多个定时线程，每隔一段时间去查一下这个Sorted Set中score小于或等于当前时间戳的元素（这可以通过**zrangebyscore**命令实现），然后再执行元素对应的任务即可。当然，执行完任务后，还要将元素从Sorted Set中删除，避免任务重复执行。如果是多个线程去轮询这个Sorted Set，还有考虑并发问题，假如说一个任务到期了，也被多个线程拿到了，这个时候必须保证只有一个线程能执行这个任务，这可以通过**zrem**命令来实现，只有删除成功了，才能执行任务，这样就能保证任务不被多个任务重复执行了。

接下来看代码。首先看下项目结构：

![图片](https://mmbiz.qpic.cn/mmbiz_png/XA3sPCPib1l6wRiaPYDicf9IZsj1bo4UZu7QOB4lwX5NDm5H04LYvGdnm3ibicMsicvjoToicUic4ZjmZf3PuZibauvOTmw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

一共4个类：Constants类定义了Redis key相关的常量。DelayTaskConsumer是延时任务的消费者，这个类负责从Redis拉取到期的任务，并封装了任务消费的逻辑。DelayTaskProducer则是延时任务的生产者，主要用于将延时任务放到Redis中。RedisClient则是Redis客户端的工具类。

最主要的类就是DelayTaskConsumer和DelayTaskProducer了。

我们先来看下生产者DelayTaskProducer：

```
public class DelayTaskProducer {

    public void produce(String newsId,long timeStamp){
        Jedis client = RedisClient.getClient();
        try {
            client.zadd(Constants.DELAY_TASK_QUEUE,timeStamp,newsId);
        }finally {
            client.close();
        }
    }

}
```

代码很简单，就是将任务描述（为了方便，这里只存储资讯的id）和任务执行的时间戳放到Redis的Sorted Set中。

接下来是延时任务的消费者DelayTaskConsumer：

```
public class DelayTaskConsumer {

    private ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();

    public void start(){
        scheduledExecutorService.scheduleWithFixedDelay(new DelayTaskHandler(),1,1, TimeUnit.SECONDS);
    }

    public static class DelayTaskHandler implements Runnable{

        @Override
        public void run() {
            Jedis client = RedisClient.getClient();
            try {
                Set<String> ids = client.zrangeByScore(Constants.DELAY_TASK_QUEUE, 0, System.currentTimeMillis(),
                        0, 1);
                if(ids==null||ids.isEmpty()){
                    return;
                }
                for(String id:ids){
                    Long count = client.zrem(Constants.DELAY_TASK_QUEUE, id);
                    if(count!=null&&count==1){
                        System.out.println(MessageFormat.format("发布资讯。id - {0} , timeStamp - {1} , " +
                                "threadName - {2}",id,System.currentTimeMillis(),Thread.currentThread().getName()));
                    }
                }
            }finally {
                client.close();
            }
        }
    }
}
```

首先看start方法。在这个方法里面我们利用Java的ScheduledExecutorService开了一个调度线程池，这个线程池会每隔1秒钟调度DelayTaskHandler中的run方法。

DelayTaskHandler类就是具体的调度逻辑了。主要有2个步骤，一个是从Redis Sorted Set中拉取到期的延时任务，另一个是执行到期的延时任务。拉取到期的延时任务是通过zrangeByScore命令实现的，处理多线程并发问题是通过zrem命令实现的。代码不复杂，这里就不多做解释了。

接下来测试一下：

```
public class DelayTaskTest {

    public static void main(String[] args) {
        DelayTaskProducer producer=new DelayTaskProducer();
        long now=new Date().getTime();
        System.out.println(MessageFormat.format("start time - {0}",now));
        producer.produce("1",now+ TimeUnit.SECONDS.toMillis(5));
        producer.produce("2",now+TimeUnit.SECONDS.toMillis(10));
        producer.produce("3",now+ TimeUnit.SECONDS.toMillis(15));
        producer.produce("4",now+TimeUnit.SECONDS.toMillis(20));
        for(int i=0;i<10;i++){
            new DelayTaskConsumer().start();
        }
    }

}
```

我们首先生产了4个延时任务，执行时间分别是程序开始运行后的5秒、10秒、15秒、20秒，然后启动了10个消费者去消费延时任务。运行效果如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/XA3sPCPib1l6wRiaPYDicf9IZsj1bo4UZu7OpibmJKpNL8wZYu69XqCr5r3OPa9uKU0X3qjCRXnSCT94E6Kn8M85oQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，任务确实能够在相应的时间点左右被执行，不过有少许时间误差，这个是因为我们拉取到期任务是通过定时任务拉取而不是实时推送的，而且拉取任务时有一部分网络开销，再者，我们的任务处理逻辑是同步处理的，需要上一次的任务处理完，才能拉取下一批任务，这些因素都会造成延时任务的执行时间产生偏差。

## 总结

以上就是通过Redis实现延时任务的思路了。这里提供的只是一个最简单的版本，实际上还有很多地方可以优化。比如，我们可以把任务的处理逻辑再放到单独的线程池中去执行，这样的话任务消费者只需要负责任务的调度就可以了，好处就是可以减少任务执行时间偏差。还有就是，这里为了方便，任务的描述存储的只是任务的id，如果有多种不同类型的任务，像前面说的发送资讯任务和推送消息任务，那么就要通过额外存储任务的类型来进行区分，或者使用不同的Sorted Set来存放延时任务了。

除此之外，上面的例子每次拉取延时任务时，只拉取1个，如果说某一个时刻要处理的任务数非常多，那么会有一部分任务延迟比较严重，这里可以优化成每次拉取不止1个到期的任务，比如说10个，然后再逐个进行处理，这样的话可以极大地提升调度效率，因为如果是使用上面的方法，拉取10个任务需要10次调度，每次间隔1秒，总共需要10秒才能把10个任务拉取完，如果改成一次拉取10个，只需要1次就能完成了，效率提升还是挺大的。

当然还可以从另一个角度来优化。大家看上面的代码，当拉取到待执行任务时，就直接执行任务，任务执行完该线程也就退出了，但是这个时候，队列里可能还有很多待执行的任务（因为我们拉取任务时，限制了拉取的数量），所以其实在这里可以使用循环，当拉取不到待执行任务时，才结束调度，当有任务时，执行完还有顺便查询下有没有堆积的任务，直到没有堆积任务了才结束线程。

最后一个需要考虑的地方是，上面的代码并没有对任务执行失败的情况进行处理，也就是说如果某个任务执行失败了，那么连重试的机会都没有。因此，在生产环境使用时，还需要考虑任务处理失败的情况。有一个简单的方法是在任务处理时捕获异常，当在处理过程中出现异常时，就将该任务再放回Redis Sorted中，或者由当前线程再重试处理。不过这样做的话，任务的时效性就不能保证了，有可能本来定在早上7点执行的任务，因为失败重试的原因，延迟到7点10分才执行完成，这个要根据业务来进行权衡，比如可以在任务描述中增加重试次数或者是否允许重试的字段，这样在任务执行失败时，就能根据不同的任务采取不同的补偿措施了。

那么使用redis实现延时任务有什么优缺点呢？优点就是可以满足吞吐量。缺点则是存在任务丢失的风险（当redis实例挂了的时候）。因此，如果对性能要求比较高，同时又能容忍少数情况下任务的丢失，那么可以使用这种方式来实现。