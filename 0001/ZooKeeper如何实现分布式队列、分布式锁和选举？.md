## 面试官：ZooKeeper如何实现分布式队列、分布式锁和选举？

ZooKeeper源码的zookeeper-recipes目录下提供了分布式队列、分布式锁和选举的实现（`GitHub地址：https://github.com/apache/zookeeper/tree/master/zookeeper-recipes`）。本文主要对这几种实现做实现原理的解析和源码剖析： 

## 分布式队列

的后缀数字是`/queue`下面现有znode最大后缀数字加1，所以该znode对应的队列元素处于队尾

```java
public class DistributedQueue {

    public boolean offer(byte[] data) throws KeeperException, InterruptedException {
        for (; ; ) {
            try {
                zookeeper.create(dir + "/" + prefix, data, acl, CreateMode.PERSISTENT_SEQUENTIAL);
                return true;
            } catch (KeeperException.NoNodeException e) {
                zookeeper.create(dir, new byte[0], acl, CreateMode.PERSISTENT);
            }
        }
    }
}
```

### 2）、element方法

```java
public class DistributedQueue {

    public byte[] element() throws NoSuchElementException, KeeperException, InterruptedException {
        Map<Long, String> orderedChildren;
        while (true) {
            try {
               //获取所有排好序的子节点
                orderedChildren = orderedChildren(null);
            } catch (KeeperException.NoNodeException e) {
                throw new NoSuchElementException();
            }
            if (orderedChildren.size() == 0) {
                throw new NoSuchElementException();
            }
      //返回队头节点的数据
            for (String headNode : orderedChildren.values()) {
                if (headNode != null) {
                    try {
                        return zookeeper.getData(dir + "/" + headNode, false, null);
                    } catch (KeeperException.NoNodeException e) {
                       //另一个客户端已经移除了队头节点,尝试获取下一个节点
                    }
                }
            }
        }
    }
  
    private Map<Long, String> orderedChildren(Watcher watcher) throws KeeperException, InterruptedException {
        Map<Long, String> orderedChildren = new TreeMap<>();

        List<String> childNames;
        childNames = zookeeper.getChildren(dir, watcher);

        for (String childName : childNames) {
            try {
                if (!childName.regionMatches(0, prefix, 0, prefix.length())) {
                    LOG.warn("Found child node with improper name: {}", childName);
                    continue;
                }
                String suffix = childName.substring(prefix.length());
                Long childId = Long.parseLong(suffix);
                orderedChildren.put(childId, childName);
            } catch (NumberFormatException e) {
                LOG.warn("Found child node with improper format : {}", childName, e);
            }
        }
        return orderedChildren;
    }  
}
```

### 3）、remove方法

```java
public class DistributedQueue {

    public byte[] remove() throws NoSuchElementException, KeeperException, InterruptedException {
        Map<Long, String> orderedChildren;
        while (true) {
            try {
               //获取所有排好序的子节点
                orderedChildren = orderedChildren(null);
            } catch (KeeperException.NoNodeException e) {
                throw new NoSuchElementException();
            }
            if (orderedChildren.size() == 0) {
                throw new NoSuchElementException();
            }
      //移除队头节点
            for (String headNode : orderedChildren.values()) {
                String path = dir + "/" + headNode;
                try {
                    byte[] data = zookeeper.getData(path, false, null);
                    zookeeper.delete(path, -1);
                    return data;
                } catch (KeeperException.NoNodeException e) {
                    //另一个客户端已经移除了队头节点,尝试移除下一个节点
                }
            }
        }
    }
}
```

## 2、分布式锁

### 1）、排他锁

排他锁的核心是如何保证当前有且仅有一个事务获取锁，并且锁被释放后，所有正在等待获取锁的事务都能够被通知到

**定义锁**

通过在ZooKeeper上创建一个子节点来表示一个锁，例如`/exclusive_lock/lock`节点就可以被定义为一个锁

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZg1TVKs1GMVRAmaFAeqttgJ6Y1sr99ictXEhFPxlyia3TysM6OXnBRUT2A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**获取锁**

在需要获取排他锁时，所有的客户端都会试图通过调用`create()`接口，在`/exclusive_lock`节点下创建临时子节点`/exclusive_lock/lock`。ZooKeeper会保证在所有的客户端中，最终只有一个客户能够创建成功，那么就可以认为该客户端获取了锁。

同时，所有没有获取到锁的客户端就需要到`/exclusive_lock`节点上注册一个子节点变更的watcher监听，以便实时监听到lock节点的变更情况

**释放锁**

`/exclusive_lock/lock`是一个临时节点，因此在以下两种情况下，都有可能释放锁

- 当前获取锁的客户端机器发生宕机，那么ZooKeeper上的这个临时节点就会被移除
- 正常执行完业务逻辑后，客户端就会主动将自己创建的临时节点删除

无论在什么情况下移除了lock节点，ZooKeeper都会通知所有在`/exclusive_lock`节点上注册了子节点变更watcher监听的客户端。这些客户端在接收到通知后，再次重新发起分布式锁获取，即重复获取锁过程

### 2）、羊群效应

上面的排他锁的实现可能引发羊群效应：当一个特定的znode改变的时候ZooKeeper触发了所有watcher的事件，由于通知的客户端很多，所以通知操作会造成ZooKeeper性能突然下降，这样会影响ZooKeeper的使用

> 改进后的分布式锁实现

**获取锁**

首先，在Zookeeper当中创建一个持久节点ParentLock。当第一个客户端想要获得锁时，需要在ParentLock这个节点下面创建一个临时顺序节点Lock1

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZg5CxlJWPhIh0dGGicqXr0p1L5y1cPjY90H5kWLRzs9FznSguWmQZQIEw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

之后，Client1查找ParentLock下面所有的临时顺序节点并排序，判断自己所创建的节点Lock1是不是顺序最靠前的一个。如果是第一个节点，则成功获得锁

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgDuIjYxdzll1BN34DKU7kgTDzvKUvkK5h0d78nmcSxycOVZibvIJwvcQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这时候，如果再有一个客户端Client2前来获取锁，则在ParentLock下再创建一个临时顺序节点Lock2

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZg5ahUiaxmiagZsxZoyCUj200mqdZ2BHrJQAcHQur6diaBwRQWCqDaNP7Rw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Client2查找ParentLock下面所有的临时顺序节点并排序，判断自己所创建的节点Lock2是不是顺序最靠前的一个，结果发现节点Lock2并不是最小的

于是，Client2向排序仅比它靠前的节点Lock1注册watcher，用于监听Lock1节点是否存在。这意味着Client2抢锁失败，进入了等待状态

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgnA0G8Z0qLYfDo65icnDgJZDw375Q1fw05FNiaW0Mdp0CJDzrVczPCbxQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这时候，如果又有一个客户端Client3前来获取锁，则在ParentLock下再创建一个临时顺序节点Lock3

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZg0AyYxLicAsicvJg62c32qm33tEXMayAqvKS6Lf6yzCt64xMpn9kdtOcQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Client3查找ParentLock下面所有的临时顺序节点并排序，判断自己所创建的节点Lock3是不是顺序最靠前的一个，结果同样发现节点Lock3并不是最小的

于是，Client3向排序仅比它靠前的节点Lock2注册watcher，用于监听Lock2节点是否存在。这意味着Client3同样抢锁失败，进入了等待状态

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgbR094ALWK9dYTtKNOd8QHCQTUfmTEvFQActBfTH1Z6dBnsKOe21dzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这样一来，Client1得到了锁，Client2监听了Lock1，Client3监听了Lock2。这恰恰形成了一个等待队列，很像是Java当中ReentrantLock所依赖的AQS

**释放锁**

释放锁分为两种情况：

1.任务完成，客户端显示释放

当任务完成时，Client1会显示调用删除节点Lock1的指令

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgWd1TRuQbSfLiaxcESmpSIibwicziaKiacXSKoZhgltFkTE7QehkhicMhEXLw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2.任务执行过程中，客户端崩溃

获得锁的Client1在任务执行过程中，如果客户端崩溃，则会断开与Zookeeper服务端的连接。根据临时节点的特性，相关联的节点Lock1会随之自动删除

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZge8vCwwkTDajmMm7wIRaIrQdR8P9FFl1wfriarvCqb1b0TUfJftpEIaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由于Client2一直监听着Lock1的存在状态，当Lock1节点被删除，Client2会立刻收到通知。这时候Client2会再次查询ParentLock下面的所有节点，确认自己创建的节点Lock2是不是目前最小的节点。如果是最小，则Client2获得了锁

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgeHibqO0dLoTKgkMdeLahnPuNiaPA43TF7D8l9vZzIDQxz0NDibEliay4eg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

同理，如果Client2也因为任务完成或者节点崩溃而删除了节点Lock2，那么Client3就会接到通知

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZg7dJoOFFrdbvTSPh3AOquRWZIiaX2pib5I38sVw4IsjqDaic84AHYhlsmA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最终，Client3成功得到了锁

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgEn7nTInFfib00m4nHvKZicJa6jBJWsQzZsHbhBdq21RjjEvXX0BibpRvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3）、共享锁

共享锁又称为读锁，在同一时刻可以允许多个线程访问，典型的就是ReentrantReadWriteLock里的读锁，它的读锁是可以被共享的，但是它的写锁确实每次只能被独占

**定义锁**

和排他锁一样，同样是通过ZooKeeper上的数据节点来表示一个锁，是一个类似于`/shared_lock/[Hostname]-请求类型-序号的临时顺序节点`，例如`/shared_lock/192.168.0.1-R-0000000001`，那么，这个节点就代表了一个共享锁，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZg5kZ9ywvO9s1HgdZJH2puJ2RWP7zsVqFNFyicbSC6hEIG8Is89pyyibrQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**获取锁**

在需要获取共享锁时，所有客户端都会到/shared_lock这个节点下面创建一个临时顺序节点，如果当前是读请求，那么就创建例如`/shared_lock/192.168.0.1-R-0000000001`的节点；如果是写请求，那么就创建例如`/shared_lock/192.168.0.1-W-0000000001`的节点

**判断读写顺序**

每个锁竞争者，只需要关注`/shared_lock`节点下序号比自己小的那个节点是否存在即可，具体实现如下：

1）客户端调用`create()`方法创建一个类似于`/shared_lock/[Hostname]-请求类型-序号的临时顺序节点`

2）客户端调用`getChildren()`接口来获取所有已经创建的子节点列表

3）判断是否可以获取共享锁：

- 读请求：没有比自己序号小的节点或者所有比自己序号小的节点都是读请求
- 写请求：序号是否最小

4）如果无法获取共享锁，那么就调用`exist()`来对比自己小的那个节点注册watcher

- 读请求：向比自己序号小的最后一个写请求节点注册watcher监听
- 写请求：向比自己序号小的最后一个节点注册watcher监听

5）等待watcher通知，继续进入步骤2

**释放锁**

释放锁的逻辑和排他锁是一致的

整个共享锁的获取和释放流程如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgiahAc3DftTRAwttjeOjX9BCO6hGTgNDedAlGrPXWvzuicvFX4WFPwoZA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4）、排他锁源码解析

**1）加锁过程**

```java
public class WriteLock extends ProtocolSupport {

    public synchronized boolean lock() throws KeeperException, InterruptedException {
        if (isClosed()) {
            return false;
        }
       //确认持久父节点是否存在
        ensurePathExists(dir);

       //真正获取锁的逻辑 调用ProtocolSupport的retryOperation()方法
        return (Boolean) retryOperation(zop);
    }
}
```

```java
class ProtocolSupport {

    protected Object retryOperation(ZooKeeperOperation operation)
        throws KeeperException, InterruptedException {
        KeeperException exception = null;
        for (int i = 0; i < RETRY_COUNT; i++) {
            try {
               //调用LockZooKeeperOperation的execute()方法
                return operation.execute();
            } catch (KeeperException.SessionExpiredException e) {
                LOG.warn("Session expired {}. Reconnecting...", zookeeper, e);
                throw e;
            } catch (KeeperException.ConnectionLossException e) {
                if (exception == null) {
                    exception = e;
                }
                LOG.debug("Attempt {} failed with connection loss. Reconnecting...", i);
                retryDelay(i);
            }
        }

        throw exception;
    }
}
```

```java
public class WriteLock extends ProtocolSupport {

    private class LockZooKeeperOperation implements ZooKeeperOperation {

        private void findPrefixInChildren(String prefix, ZooKeeper zookeeper, String dir)
            throws KeeperException, InterruptedException {
            List<String> names = zookeeper.getChildren(dir, false);
            for (String name : names) {
                if (name.startsWith(prefix)) {
                    id = name;
                    LOG.debug("Found id created last time: {}", id);
                    break;
                }
            }
            if (id == null) {
                id = zookeeper.create(dir + "/" + prefix, data, getAcl(), EPHEMERAL_SEQUENTIAL);

                LOG.debug("Created id: {}", id);
            }

        }

        @SuppressFBWarnings(
            value = "NP_NULL_PARAM_DEREF_NONVIRTUAL",
            justification = "findPrefixInChildren will assign a value to this.id")
        public boolean execute() throws KeeperException, InterruptedException {
            do {
                if (id == null) {
                    long sessionId = zookeeper.getSessionId();
                    String prefix = "x-" + sessionId + "-";
                   //创建临时顺序节点
                    findPrefixInChildren(prefix, zookeeper, dir);
                    idName = new ZNodeName(id);
                }
               //获取所有子节点
                List<String> names = zookeeper.getChildren(dir, false);
                if (names.isEmpty()) {
                    LOG.warn("No children in: {} when we've just created one! Lets recreate it...", dir);
                    id = null;
                } else {
                   //对所有子节点进行排序
                    SortedSet<ZNodeName> sortedNames = new TreeSet<>();
                    for (String name : names) {
                        sortedNames.add(new ZNodeName(dir + "/" + name));
                    }
                    ownerId = sortedNames.first().getName();
                    SortedSet<ZNodeName> lessThanMe = sortedNames.headSet(idName);
                   //是否存在序号比自己小的节点
                    if (!lessThanMe.isEmpty()) {
                        ZNodeName lastChildName = lessThanMe.last();
                        lastChildId = lastChildName.getName();
                        LOG.debug("Watching less than me node: {}", lastChildId);
                       //有序号比自己小的节点,则调用exist()向前一个节点注册watcher
                        Stat stat = zookeeper.exists(lastChildId, new LockWatcher());
                        if (stat != null) {
                            return Boolean.FALSE;
                        } else {
                            LOG.warn("Could not find the stats for less than me: {}", lastChildName.getName());
                        }
                    } 
                   //没有序号比自己小的节点,则获取锁
                   else {
                        if (isOwner()) {
                            LockListener lockListener = getLockListener();
                            if (lockListener != null) {
                                lockListener.lockAcquired();
                            }
                            return Boolean.TRUE;
                        }
                    }
                }
            }
            while (id == null);
            return Boolean.FALSE;
        }
    }
```

**2）解锁过程**

```java
public class WriteLock extends ProtocolSupport {

    public synchronized void unlock() throws RuntimeException {

        if (!isClosed() && id != null) {
            try {
        //删除当前节点,此时会触发后一个节点的watcher
                ZooKeeperOperation zopdel = () -> {
                    zookeeper.delete(id, -1);
                    return Boolean.TRUE;
                };
                zopdel.execute();
            } catch (InterruptedException e) {
                LOG.warn("Unexpected exception", e);
                Thread.currentThread().interrupt();
            } catch (KeeperException.NoNodeException e) {
            } catch (KeeperException e) {
                LOG.warn("Unexpected exception", e);
                throw new RuntimeException(e.getMessage(), e);
            } finally {
                LockListener lockListener = getLockListener();
                if (lockListener != null) {
                    lockListener.lockReleased();
                }
                id = null;
            }
        }
    }
}
```

## 3、选举

使用临时顺序znode来表示选举请求，创建最小后缀数字znode的选举请求成功。在协同设计上和分布式锁是一样的，不同之处在于具体实现。不同于分布式锁，选举的具体实现对选举的各个阶段做了细致的监控

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucEPufGZNeicOOYFa4NbSmZgWYs1iag61xtFcwUtpaspMVXSHeLd57Xo2OLe6mmCg2aQuLXMwcQdNLA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

```java
public class LeaderElectionSupport implements Watcher {    

    public synchronized void start() {
        state = State.START;
        dispatchEvent(EventType.START);

        LOG.info("Starting leader election support");

        if (zooKeeper == null) {
            throw new IllegalStateException(
                "No instance of zookeeper provided. Hint: use setZooKeeper()");
        }

        if (hostName == null) {
            throw new IllegalStateException(
                "No hostname provided. Hint: use setHostName()");
        }

        try {
           //发起选举请求 创建临时顺序节点
            makeOffer();
           //选举请求是否被满足
            determineElectionStatus();
        } catch (KeeperException | InterruptedException e) {
            becomeFailed(e);
        }
    }
  
    private void makeOffer() throws KeeperException, InterruptedException {
        state = State.OFFER;
        dispatchEvent(EventType.OFFER_START);

        LeaderOffer newLeaderOffer = new LeaderOffer();
        byte[] hostnameBytes;
        synchronized (this) {
            newLeaderOffer.setHostName(hostName);
            hostnameBytes = hostName.getBytes();
            newLeaderOffer.setNodePath(zooKeeper.create(rootNodeName + "/" + "n_",
                                                        hostnameBytes, ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                                        CreateMode.EPHEMERAL_SEQUENTIAL));
            leaderOffer = newLeaderOffer;
        }
        LOG.debug("Created leader offer {}", leaderOffer);

        dispatchEvent(EventType.OFFER_COMPLETE);
    }
  
    private void determineElectionStatus() throws KeeperException, InterruptedException {

        state = State.DETERMINE;
        dispatchEvent(EventType.DETERMINE_START);

        LeaderOffer currentLeaderOffer = getLeaderOffer();

        String[] components = currentLeaderOffer.getNodePath().split("/");

        currentLeaderOffer.setId(Integer.valueOf(components[components.length - 1].substring("n_".length())));
    //获取所有子节点并排序
        List<LeaderOffer> leaderOffers = toLeaderOffers(zooKeeper.getChildren(rootNodeName, false));

        for (int i = 0; i < leaderOffers.size(); i++) {
            LeaderOffer leaderOffer = leaderOffers.get(i);

            if (leaderOffer.getId().equals(currentLeaderOffer.getId())) {
                LOG.debug("There are {} leader offers. I am {} in line.", leaderOffers.size(), i);

                dispatchEvent(EventType.DETERMINE_COMPLETE);
        
               //如果当前节点是第一个,则成为Leader
                if (i == 0) {
                    becomeLeader();
                } 
               //如果有选举请求在当前节点前面,则进行等待,调用exist()向前一个节点注册watcher
               else {
                    becomeReady(leaderOffers.get(i - 1));
                }
                break;
            }
        }
    }
}
```





