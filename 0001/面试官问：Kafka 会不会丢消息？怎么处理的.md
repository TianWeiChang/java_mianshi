面试官问：Kafka 会不会丢消息？怎么处理的?

Kafka存在丢消息的问题，消息丢失会发生在Broker，Producer和Consumer三种。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/8Jeic82Or04nIqNre5oDC52VXM9ibclju9eqcAXcrtsmhHjxNzHajE3jBrE96KxgEF8eJxbguYyAd8VicOfgxw5iaw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## Broker

Broker丢失消息是由于Kafka本身的原因造成的，kafka为了得到更高的性能和吞吐量，将数据异步批量的存储在磁盘中。消息的刷盘过程，为了提高性能，减少刷盘次数，kafka采用了批量刷盘的做法。即，按照一定的消息量，和时间间隔进行刷盘。这种机制也是由于linux操作系统决定的。将数据存储到linux操作系统种，会先存储到页缓存（Page cache）中，按照时间或者其他条件进行刷盘（从page cache到file），或者通过fsync命令强制刷盘。数据在page cache中时，如果系统挂掉，数据会丢失。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04nIqNre5oDC52VXM9ibclju939lGdrianTEAyWMZXBI5oymCJaL0aLSaUlF4vjJ74gCjmaQgDVNicWOQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Broker在linux服务器上高速读写以及同步到Replica

上图简述了broker写数据以及同步的一个过程。broker写数据只写到PageCache中，而pageCache位于内存。这部分数据在断电后是会丢失的。pageCache的数据通过linux的flusher程序进行刷盘。刷盘触发条件有三：

- 主动调用sync或fsync函数
- 可用内存低于阀值
- dirty data时间达到阀值。dirty是pagecache的一个标识位，当有数据写入到pageCache时，pagecache被标注为dirty，数据刷盘以后，dirty标志清除。

Broker配置刷盘机制，是通过调用fsync函数接管了刷盘动作。从单个Broker来看，pageCache的数据会丢失。

Kafka没有提供同步刷盘的方式。同步刷盘在RocketMQ中有实现，实现原理是将异步刷盘的流程进行阻塞，等待响应，类似ajax的callback或者是java的future。下面是一段rocketmq的源码。

```
GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
service.putRequest(request);
boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout()); // 刷盘
```

也就是说，理论上，要完全让kafka保证单个broker不丢失消息是做不到的，只能通过调整刷盘机制的参数缓解该情况。比如，减少刷盘间隔，减少刷盘数据量大小。时间越短，性能越差，可靠性越好（尽可能可靠）。这是一个选择题。

为了解决该问题，kafka通过producer和broker协同处理单个broker丢失参数的情况。一旦producer发现broker消息丢失，即可自动进行retry。除非retry次数超过阀值（可配置），消息才会丢失。此时需要生产者客户端手动处理该情况。那么producer是如何检测到数据丢失的呢？是通过ack机制，类似于http的三次握手的方式。

> The number of acknowledgments the producer requires the leader to have received before considering a request complete. This controls the durability of records that are sent. The following settings are allowed: acks=0 If set to zero then the producer will not wait for any acknowledgment from the server at all. The record will be immediately added to the socket buffer and considered sent. No guarantee can be made that the server has received the record in this case, and the retries configuration will not take effect (as the client won’t generally know of any failures). The offset given back for each record will always be set to -1. acks=1 This will mean the leader will write the record to its local log but will respond without awaiting full acknowledgement from all followers. In this case should the leader fail immediately after acknowledging the record but before the followers have replicated it then the record will be lost. acks=allThis means the leader will wait for the full set of in-sync replicas to acknowledge the record. This guarantees that the record will not be lost as long as at least one in-sync replica remains alive. This is the strongest available guarantee. This is equivalent to the acks=-1 setting.

以上的引用是kafka官方对于参数acks的解释（在老版本中，该参数是request.required.acks）。

- acks=0，producer不等待broker的响应，效率最高，但是消息很可能会丢。

- acks=1，leader broker收到消息后，不等待其他follower的响应，即返回ack。也可以理解为ack数为1。此时，如果follower还没有收到leader同步的消息leader就挂了，那么消息会丢失。按照上图中的例子，如果leader收到消息，成功写入PageCache后，会返回ack，此时producer认为消息发送成功。但此时，按照上图，数据还没有被同步到follower。如果此时leader断电，数据会丢失。

- acks=-1，leader broker收到消息后，挂起，等待所有ISR列表中的follower返回结果后，再返回ack。-1等效与all。这种配置下，只有leader写入数据到pagecache是不会返回ack的，还需要所有的ISR返回“成功”才会触发ack。如果此时断电，producer可以知道消息没有被发送成功，将会重新发送。如果在follower收到数据以后，成功返回ack，leader断电，数据将存在于原来的follower中。在重新选举以后，新的leader会持有该部分数据。数据从leader同步到follower，需要2步：

- - 数据从pageCache被刷盘到disk。因为只有disk中的数据才能被同步到replica。
  - 数据同步到replica，并且replica成功将数据写入PageCache。在producer得到ack后，哪怕是所有机器都停电，数据也至少会存在于leader的磁盘内。

上面第三点提到了ISR的列表的follower，需要配合另一个参数才能更好的保证ack的有效性。ISR是Broker维护的一个“可靠的follower列表”，in-sync Replica列表，broker的配置包含一个参数：min.insync.replicas。该参数表示ISR中最少的副本数。如果不设置该值，ISR中的follower列表可能为空。此时相当于acks=1。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04nIqNre5oDC52VXM9ibclju9pqQIKwlW65Q7d6w7S6QCrmTTZbtok6icA2XAzv3GCeFoA71k2fITAGQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图中：

- acks=0，总耗时f(t) = f(1)。
- acks=1，总耗时f(t) = f(1) + f(2)。
- acks=-1，总耗时f(t) = f(1) + max( f(A) , f(B) ) + f(2)。

性能依次递减，可靠性依次升高。

## Producer

Producer丢失消息，发生在生产者客户端。

为了提升效率，减少IO，producer在发送数据时可以将多个请求进行合并后发送。被合并的请求咋发送一线缓存在本地buffer中。缓存的方式和前文提到的刷盘类似，producer可以将请求打包成“块”或者按照时间间隔，将buffer中的数据发出。通过buffer我们可以将生产者改造为异步的方式，而这可以提升我们的发送效率。

但是，buffer中的数据就是危险的。在正常情况下，客户端的异步调用可以通过callback来处理消息发送失败或者超时的情况，但是，一旦producer被非法的停止了，那么buffer中的数据将丢失，broker将无法收到该部分数据。又或者，当Producer客户端内存不够时，如果采取的策略是丢弃消息（另一种策略是block阻塞），消息也会被丢失。抑或，消息产生（异步产生）过快，导致挂起线程过多，内存不足，导致程序崩溃，消息丢失。

![图片](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04nIqNre5oDC52VXM9ibclju9niaUX0T1yGh6UVFRicyQ5eljhtiaYR9408LpzXEsamQEXtcJxkfHA3jGQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

producer采取批量发送的示意图

![图片](https://mmbiz.qpic.cn/mmbiz_png/8Jeic82Or04nIqNre5oDC52VXM9ibclju9Prg4KNbJ6JNWsksV3nCUgmxX9Lx7MciaqYYsqYictuql9NqmwzIvZjwA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

异步发送消息生产速度过快的示意图

根据上图，可以想到几个解决的思路：

- 异步发送消息改为同步发送消。或者service产生消息时，使用阻塞的线程池，并且线程数有一定上限。整体思路是控制消息产生速度。
- 扩大Buffer的容量配置。这种方式可以缓解该情况的出现，但不能杜绝。
- service不直接将消息发送到buffer（内存），而是将消息写到本地的磁盘中（数据库或者文件），由另一个（或少量）生产线程进行消息发送。相当于是在buffer和service之间又加了一层空间更加富裕的缓冲层。

## Consumer

Consumer消费消息有下面几个步骤：

- 接收消息
- 处理消息
- 反馈“处理完毕”（commited）

Consumer的消费方式主要分为两种：

- 自动提交offset，Automatic Offset Committing
- 手动提交offset，Manual Offset Control

Consumer自动提交的机制是根据一定的时间间隔，将收到的消息进行commit。commit过程和消费消息的过程是异步的。也就是说，可能存在消费过程未成功（比如抛出异常），commit消息已经提交了。此时消息就丢失了。

```
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "test");
// 自动提交开关
props.put("enable.auto.commit", "true");
// 自动提交的时间间隔，此处是1s
props.put("auto.commit.interval.ms", "1000");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("foo", "bar"));
while (true) {
        // 调用poll后，1000ms后，消息状态会被改为 committed
  ConsumerRecords<String, String> records = consumer.poll(100);
  for (ConsumerRecord<String, String> record : records)
    insertIntoDB(record); // 将消息入库，时间可能会超过1000ms
```

上面的示例是自动提交的例子。如果此时，`insertIntoDB(record)`发生异常，消息将会出现丢失。接下来是手动提交的例子：

```
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "test");
// 关闭自动提交，改为手动提交
props.put("enable.auto.commit", "false");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("foo", "bar"));
final int minBatchSize = 200;
List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
while (true) {
        // 调用poll后，不会进行auto commit
  ConsumerRecords<String, String> records = consumer.poll(100);
  for (ConsumerRecord<String, String> record : records) {
    buffer.add(record);
  }
  if (buffer.size() >= minBatchSize) {
    insertIntoDb(buffer);
                // 所有消息消费完毕以后，才进行commit操作
    consumer.commitSync();
    buffer.clear();
  }
```

将提交类型改为手动以后，可以保证消息“至少被消费一次”(at least once)。但此时可能出现重复消费的情况，重复消费不属于本篇讨论范围。

上面两个例子，是直接使用Consumer的High level API，客户端对于offset等控制是透明的。也可以采用Low level API的方式，手动控制offset，也可以保证消息不丢，不过会更加复杂。

```
 try {
     while(running) {
         ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
         for (TopicPartition partition : records.partitions()) {
             List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
             for (ConsumerRecord<String, String> record : partitionRecords) {
                 System.out.println(record.offset() + ": " + record.value());
             }
             long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
             // 精确控制offset
             consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastOffset + 1)));
         }
     }
 } finally {
   consumer.close();
 }
```

asdasd