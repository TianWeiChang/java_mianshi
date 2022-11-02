细说Redis分布式锁

##  **序-碎碎叨叨**

在家办公的第N周，

也不知道笔者工位上的键盘和显示器有没有想我，

不知道会不会落灰太严重，被保洁阿姨扔掉了。

笔者今天带来一篇关于redis锁的文章。

**连敲带画**码出此文，有一些细节，对redis锁不清晰的盆友不妨瞧一瞧。

如果是有经验的盆友，挑挑毛病，那笔者是更感谢了～

闲话不多，马上发车。

## 正文-开门见山

谈起redis锁，下面三个，算是出现最多的高频词汇：

- setnx
- redLock
- redisson

### setnx

其实目前通常所说的setnx命令，并非单指redis的setnx key value这条命令。

一般代指redis中对**set**命令加上**nx**参数进行使用， set这个命令，目前已经支持这么多参数可选：

> `SET key value [EX seconds|PX milliseconds][NX|XX] [KEEPTTL]`

当然了，就不在文章中默写Api了，基础参数还有不清晰的，可以蹦到官网。

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1DOL13qxeia5GMXfjgtPwDhCbJfeBPSZAfTS8TUZBiaicXdDOcpUhJDQovA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图是笔者画的**setnx**大致原理，主要依托了它的**key不存在才能set成功的特性**，进程A拿到锁，在没有删除锁的Key时，进程B自然获取锁就失败了。

那么为什么要使用PX 30000去设置一个超时时间？

是怕**进****程A**不讲道理啊，锁没等释放呢，**万一崩了**，直接原地把锁带走了，导致系统中谁也拿不到锁。

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1DNiaHEoTlaFxQSOuQLof8SEkLfB5uzAD0E7CQhibibKPphOtkgHU6rqXnQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

就算这样，还是不能保证万无一失。

如果进程A又不讲道理，操作锁内资源超过笔者设置的超时时间，那么就会导致其他进程拿到锁，等进程A回来了，回手就是把其他进程的锁删了，如图：

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1DkyeqvGnXzX9p5PibsaHRBP4Kjcq6oNvicAeAWZ9DDW2npoxfibIRBdlcA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

还是刚才那张图，将**T5**时刻改成了锁超时，被redis释放。

**进程B**在**T6**开开心心拿到锁不到一会，**进程A**操作完成，回手一个**del**，就把锁释放了。

当**进程B**操作完成，去释放锁的时候（图中T8时刻）：

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1D70wFB8CNwDHjKS9l2Ib3SPZdfO0Bwvm2XZDHT3rQzLaZMdrLf0DvMg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

找不到锁其实还算好的，万一**T7**时刻有个**进程C**过来加锁成功，那么**进程B**就把**进程C**的锁释放了。
以此类推，**进程C**可能释放**进程D**的锁，**进程D**....(禁止套娃)，具体什么后果就不得而知了。

所以在用**setnx**的时候，key虽然是主要作用，但是**value**也不能闲着，可以设置一个**唯一的客户端ID**，或者用**UUID**这种随机数。

当解锁的时候，先获取value判断是否是当前进程加的锁，再去删除。**伪代码：**

```
String uuid = xxxx;
// 伪代码，具体实现看项目中用的连接工具
// 有的提供的方法名为set 有的叫setIfAbsent
set Test uuid NX PX 3000
try{
// biz handle....
} finally {
    // unlock
    if(uuid.equals(redisTool.get('Test')){
        redisTool.del('Test');
    }
}
```

这回看起来是不是稳了。

相反，这回的问题更明显了，在finally代码块中，**get和del并非原子操作**，还是有进程安全问题。

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1Dr2G3ZxBs7p2xHSLuGMWfGjzLqOqVefbEMIbJsZOib1J7MlwibOoJ4kkg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为什么有问题还说这么多呢？

第一，搞清劣势所在，才能更好的完善。

第二点，其实上文中最后这段代码，还是有**很多公司在用**的。

*大小项目悖论：大公司实现规范，但是小司小项目虽然存在不严谨，可并发倒也不高，出问题的概率和大公司一样低。-- 鲁迅*

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1DBT13Om9CVuia0pe3R23GlF9wIYoYiaM7ryAtuQXDKQUx2UXajPf0GDvg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那么删除锁的正确姿势之一，就是可以使用**lua**脚本，通过redis的**eval/evalsha**命令来运行：

```lua
-- lua删除锁：
-- KEYS和ARGV分别是以集合方式传入的参数，对应上文的Test和uuid。
-- 如果对应的value等于传入的uuid。
if redis.call('get', KEYS[1]) == ARGV[1] 
    then 
	-- 执行删除操作
        return redis.call('del', KEYS[1]) 
    else 
	-- 不成功，返回0
        return 0 
end
```

通过**lua**脚本能保证原子性的原因说的通俗一点：

就算你在**lua**里写出花，执行也是一个命令(**eval/evalsha**)去执行的，一条命令没执行完，其他客户端是看不到的。

那么既然这么麻烦，有没有比较好的工具呢？就要说到**redisson**了。

介绍**redisson**之前，笔者简单解释一下为什么现在的**setnx**默认是指**set**命令带上**nx**参数，而不是直接说是**setnx**这个命令。

因为redis版本在**2.6.12**之前，set是不支持nx参数的，如果想要完成一个锁，那么需要两条命令：

```
1. setnx Test uuid
2. expire Test 30
```

即放入Key和设置有效期，是分开的两步，理论上会出现1刚执行完，程序挂掉，无法保证原子性。

但是早在**2013年**，也就是7年前，Redis就发布了**2.6.12**版本，并且官网(set命令页)，也早早就说明了“SETNX, SETEX, PSETEX可能在未来的版本中，会弃用并**永久删除**”。

笔者曾阅读过一位大佬的文章，其中就有一句指导入门者的面试小套路，具体文字忘记了，大概意思如下：

*说到redis锁的时候，可以先从setnx讲起，最后慢慢引出set命令的可以加参数，可以体现出自己的知识面。*

如果有缘你也阅读过这篇文章，并且学到了这个套路，作为本文的笔者我要加一句提醒：

请注意你的工作年限！首先回答官网表明即将废弃的命令，再引出set命令七年前的“新特性”，如果是刚毕业不久的人这么说，面试官会以为自己穿越了。

*你套路面试官，面试官也会套路你。-- vt・沃兹基硕德*

### **|** **redisson**

Redisson是**java的redis客户端之一**，提供了一些api方便操作redis。

但是redisson这个客户端可有点厉害，笔者在官网截了仅仅是一部分的图：

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1Db0G9Vp0pribZ5FiaKPEY2tDlkibCTEE1ptQ1IOoSa4ZqZsibQ1eWCqNxHA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个特性列表可以说是太多了，是不是还看到了一些**JUC**包下面的类名，redisson帮我们搞了分布式的版本，比如**AtomicLong**，直接用**RedissonAtomicLong**就行了，连类名都不用去新记，很人性化了。

锁只是它的冰山一角，并且从它的wiki页面看到，对主从，哨兵，集群等模式都支持，当然了，单节点模式肯定是支持的。

本文还是以锁为主，其他的不过多介绍。

Redisson普通的锁实现源码主要是**RedissonLock**这个类，还没有看过它源码的盆友，不妨去瞧一瞧。

源码中**加锁/释放锁**操作都是用**lua**脚本完成的，封装的非常完善，开箱即用。

这里有个小细节，加锁使用**setnx**就能实现，也采用**lua**脚本是不是多此一举？笔者也**非常严谨**的思考了一下：这么厉害的东西哪能写废代码？

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1Dwt2vcEMdK27IItTx7vaeLJO8RxIqyEpDiauia7GZqBIvqvUcIpM4wBNQ/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其实笔者仔细看了一下，加锁解锁的lua脚本考虑的非常全面，其中就包括锁的重入性，这点可以说是考虑非常周全，我也随手写了代码测试一下：

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1Dicx6UZic7ibBOPsibJPL2bL4I7oLvzgr1djWQAKGGS3hrrPxJ4q4Ah1pFA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

的确用起来像jdk的**ReentrantLock**一样丝滑，那么redisson实现的已经这么完善，redLock又是什么？

### **|** **RedLock**

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1DT6ka5GwhoSZzTmwLQaD6swufgvKpbPhnoddTwiaLdC5XMcLDNUAjHFw/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**redLock**的中文是直译过来的，就叫**红锁**。

红锁并非是一个工具，而是redis官方提出的一种分布式锁的**算法**。

就在刚刚介绍完的redisson中，就实现了redLock版本的锁。也就是说除了**getLock**方法，还有**getRedLock**方法。

笔者大概画了一下对红锁的理解：

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLZsiaEibyjwLptGrib7iaA3ag1D4RUvIzLLpUStvr92rAjQeOuB82w81dFGydQyEJLl55vgDg2ibVTOnPw/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果你不熟悉redis高可用部署，那么没关系。redLock算法虽然是需要多个实例，但是这些实例都是独自部署的，没有主从关系。

RedLock作者指出，之所以要用独立的，是避免了redis异步复制造成的锁丢失，比如：主节点没来的及把**刚刚set进来这条数**据给从节点，就挂了。

有些人是不是觉得大佬们都是杠精啊，天天就想着极端情况。其实**高可用**嘛，拼的就是**99.999...%** 中小数点后面的位数。

回到上面那张简陋的图片，红锁算法认为，只要(N/2) + 1个节点加锁成功，那么就认为获取了锁， 解锁时将所有实例解锁。流程为：

1. 顺序向五个节点请求加锁
2. 根据一定的**超时时间**来推断是不是跳过该节点
3. 三个节点加锁成功并且花费时间小于锁的有效期
4. 认定加锁成功

也就是说，假设锁**30秒**过期，三个节点加锁花了31秒，自然是加锁失败了。

这只是举个例子，实际上并不应该等每个节点那么长时间，就像官网所说的那样，假设有效期是**1****0秒**，那么单个redis实例操作超时时间，应该在**5**到**50毫****秒**(注意时间单位)。

还是假设我们设置有效期是30秒，图中超时了两个redis节点。
那么加锁成功的节点**总共花费了3秒**，所以锁的**实际有效**期是小于27秒的。

即扣除加锁成功三个实例的3秒，还要扣除等待超时redis实例的总共时间。

看到这，你有可能对这个算法有一些疑问，那么你不是一个人。

回头看看Redis官网关于红锁的描述。

就在这篇描述页面的最下面，你能看到著名的关于红锁的**神仙打架**事件。

![图片](E:\other\网络\assets\1632026444021.png)

即Martin Kleppmann和antirez的redLock辩论. 一个是很有资历的分布式架构师，一个是redis之父。

官方挂人，最为致命。

开个玩笑，要是质疑能被官方挂到官网，说明肯定是有价值的。

所以说如果项目里要使用红锁，除了红锁的介绍，不妨要多看两篇文章，即：

1. Martin Kleppmann的质疑贴
2. antirez的反击贴

## **|** **总 结**

看了这么多，是不是发现如何实现，都不能保证100%的稳定。

程序就是这样，没有绝对的稳定，所以做好人工补偿环节也是重要的一环，毕竟：

技术不够，人工来凑～
