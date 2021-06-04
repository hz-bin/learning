
## 什么是Zookeeper
https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653193977&idx=1&sn=12319f8cb81f55a40ac461bd0ad9d74e&chksm=8c99f423bbee7d35056ce7ec1b321f33aad15c309de6eba0086cb31a48b975ccb1d695d5a251&scene=21#wechat_redirect
- Zookeeper是一个分布式协调服务，可以在分布式系统中共享配置，协调锁资源，提供命名服务。
- Zookeeper的数据模型很像数据结构当中的树，也很像文件系统的目录。
- Zookeeper的数据存储基于节点，这种节点叫做Znode。Znode包含了数据、子节点引用、访问权限等等。
    - data：Znode存储的数据信息。
    - ACL：记录Znode的访问权限，即哪些人或哪些IP可以访问本节点。
    - stat：包含Znode的各种元数据，比如事务ID、版本号、时间戳、大小等等。
    - child：当前节点的子节点引用，类似于二叉树的左孩子右孩子。
- Zookeeper是为读多写少的场景所设计。Znode并不是用来存储大规模业务数据，而是用于存储少量的状态和配置信息，每个节点的数据最大不能超过1MB。
- Zookeeper包含的基本操作：
    - create：创建节点
    - delete：删除节点
    - exists：判断节点是否存在
    - getData：获得一个节点的数据
    - setData：设置一个节点的数据
    - getChildren：获取节点下的所有子节点
- Zookeeper客户端在请求读操作的时候，可以选择是否设置Watch。Watch可以理解成是注册在特定Znode上的触发器。当这个Znode发生改变，也就是调用了create，delete，setData方法的时候，将会触发Znode上注册的对应事件，请求Watch的客户端会接收到异步通知。

### Zookeeper的一致性
- Zookeeper Service集群是一主多从结构。
- 在更新数据时，首先更新到主节点（这里的节点是指服务器，不是Znode），再同步到从节点。
- 在读取数据时，直接读取任意从节点。
- 为了保证主从节点的数据一致性，Zookeeper采用了ZAB（ZooKeeper Atomic Broadcast）协议，这种协议非常类似于一致性算法Paxos和Raft。
- ZAB协议所定义的三种节点状态：
    - Looking ：选举状态。
    - Following ：Follower节点（从节点）所处的状态。
    - Leading ：Leader节点（主节点）所处状态。
- 最大ZXID也就是节点本地的最新事务编号，包含epoch和计数两部分。epoch是纪元的意思，相当于Raft算法选主时候的term。

#### ZAB的崩溃恢复
- 假如Zookeeper当前的主节点挂掉了，集群会进行崩溃恢复。ZAB的崩溃恢复分成三个阶段：

##### 1、Leader election
- 选举阶段，此时集群中的节点处于Looking状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器ID和最新事务ID（ZXID）。
- 接下来，节点会用自身的ZXID和从其他节点接收到的ZXID做比较，如果发现别人家的ZXID比自己大，也就是数据比自己新，那么就重新发起投票，投票给目前已知最大的ZXID所属节点。
- 每次投票后，服务器都会统计投票数量，判断是否有某个节点得到半数以上的投票。如果存在这样的节点，该节点将会成为准Leader，状态变为Leading。其他节点的状态变为Following。

##### 2、Discovery
- 发现阶段，用于在从节点中发现最新的ZXID和事务日志。或许有人会问：既然Leader被选为主节点，已经是集群里数据最新的了，为什么还要从节点中寻找最新事务呢？
- 这是为了防止某些意外情况，比如因网络原因在上一阶段产生多个Leader的情况。
- 所以这一阶段，Leader集思广益，接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。
- 各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。

##### 3.Synchronization
- 同步阶段，把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。
- 自此，故障恢复正式完成。

#### Broadcast
- 什么是Broadcast呢？简单来说，就是Zookeeper常规情况下更新数据的时候，由Leader广播到所有的Follower。其过程如下：
- 1.客户端发出写入数据请求给任意Follower。
- 2.Follower把写入数据请求转发给Leader。
- 3.Leader采用二阶段提交方式，先发送Propose广播给Follower。
- 4.Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。
- 5.Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。

Zab协议既不是强一致性，也不是弱一致性，而是处于两者之间的单调一致性。它依靠事务ID和版本号，保证了数据的更新和读取是有序的。


### Zookeeper的应用
- 1.分布式锁：这是雅虎研究员设计Zookeeper的初衷。利用Zookeeper的临时顺序节点，可以轻松实现分布式锁。
- 2.服务注册和发现：利用Znode和Watcher，可以实现分布式服务的注册和发现。最著名的应用就是阿里的分布式RPC框架Dubbo。
- 3.共享配置和状态信息：Redis的分布式解决方案Codis，就利用了Zookeeper来存放数据路由表和 codis-proxy 节点的元信息。同时 codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy。
- 此外，Kafka、HBase、Hadoop，也都依靠Zookeeper同步节点信息，实现高可用。


## 什么是分布式锁
http://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653194065&idx=1&sn=1baa162e40d48ce9b44ea5c4b2c71ad7&chksm=8c99f58bbbee7c9d5b5725da5ee38fe0f89d7a816f3414806785aea0fe5ae766769600d3e982&scene=21#wechat_redirect
- Redis分布式锁：`set keyname val ex 5 nx`，当keyname不存在时，设置key，过期时间是5秒

## 如何用Zookeeper实现分布式锁
http://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653194140&idx=1&sn=07b65a50798c26ecdc0fc555128ab937&chksm=8c99f546bbee7c50b1642dc971cb1f5e244dce661546e141734797c8c23c6c3ad779dfb57d3b&scene=21#wechat_redirect

### Znode分为四种类型：
- 1.持久节点 （PERSISTENT）：默认的节点类型。创建节点的客户端与zookeeper断开连接后，该节点依旧存在 。
- 2.持久节点顺序节点（PERSISTENT_SEQUENTIAL）：所谓顺序节点，就是在创建节点时，Zookeeper根据创建的时间顺序给该节点名称进行编号
- 3.临时节点（EPHEMERAL）：和持久节点相反，当创建节点的客户端与zookeeper断开连接后，临时节点会被删除。
- 4.临时顺序节点（EPHEMERAL_SEQUENTIAL）：临时顺序节点结合和临时节点和顺序节点的特点：在创建节点时，Zookeeper根据创建的时间顺序给该节点名称进行编号；当创建节点的客户端与zookeeper断开连接后，临时节点会被删除。

### Zookeeper分布式锁的原理
Zookeeper分布式锁恰恰应用了临时顺序节点。具体如何实现呢？让我们来看一看详细步骤：

#### 获取锁
首先，在Zookeeper当中创建一个持久节点ParentLock。当第一个客户端想要获得锁时，需要在ParentLock这个节点下面创建一个临时顺序节点 Lock1。
之后，Client1查找ParentLock下面所有的临时顺序节点并排序，判断自己所创建的节点Lock1是不是顺序最靠前的一个。如果是第一个节点，则成功获得锁。
这时候，如果再有一个客户端 Client2 前来获取锁，则在ParentLock下载再创建一个临时顺序节点Lock2。
Client2查找ParentLock下面所有的临时顺序节点并排序，判断自己所创建的节点Lock2是不是顺序最靠前的一个，结果发现节点Lock2并不是最小的。
于是，Client2向排序仅比它靠前的节点Lock1注册Watcher，用于监听Lock1节点是否存在。这意味着Client2抢锁失败，进入了等待状态。
这时候，如果又有一个客户端Client3前来获取锁，则在ParentLock下载再创建一个临时顺序节点Lock3。
Client3查找ParentLock下面所有的临时顺序节点并排序，判断自己所创建的节点Lock3是不是顺序最靠前的一个，结果同样发现节点Lock3并不是最小的。
于是，Client3向排序仅比它靠前的节点Lock2注册Watcher，用于监听Lock2节点是否存在。这意味着Client3同样抢锁失败，进入了等待状态。
这样一来，Client1得到了锁，Client2监听了Lock1，Client3监听了Lock2。这恰恰形成了一个等待队列，很像是Java当中ReentrantLock所依赖的AQS（AbstractQueuedSynchronizer）。

#### 释放锁
释放锁分为两种情况：
1.任务完成，客户端显示释放
当任务完成时，Client1会显示调用删除节点Lock1的指令。
2.任务执行过程中，客户端崩溃
获得锁的Client1在任务执行过程中，如果Duang的一声崩溃，则会断开与Zookeeper服务端的链接。根据临时节点的特性，相关联的节点Lock1会随之自动删除。
由于Client2一直监听着Lock1的存在状态，当Lock1节点被删除，Client2会立刻收到通知。这时候Client2会再次查询ParentLock下面的所有节点，确认自己创建的节点Lock2是不是目前最小的节点。如果是最小，则Client2顺理成章获得了锁。
同理，如果Client2也因为任务完成或者节点崩溃而删除了节点Lock2，那么Client3就会接到通知。
最终，Client3成功得到了锁。


## Zookeeper如何保证数据一致性
https://juejin.im/post/6844904163042656263


## Kafka
- https://www.jianshu.com/p/5f510cb9d7f1

- https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/message-queue/Kafka%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93