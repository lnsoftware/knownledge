# ZAB协议



## 概念

ZAB（原子消息广播协议）。在Zookeeper中，主要依赖ZAB协议来**实现分布式数据一致性**。

Zookeeper使用一个单一的主进程来接收并处理所有客户端的**事务请求（也就是一个请求，如果说HTTP请求）**，并采用ZAB的原子广播协议，将服务器的数据状态变更以**事务提议**的方式广播到所有的副本进程上。



**ZAB的核心是定义了事务请求的处理方式，即**

- 所有事务请求都必须由一个全局唯一的服务器来协调处理。这样的服务器被称为Leader服务器，而余下的其他服务器则称为Follower服务器。
- Leader服务器负责将事务请求转化为事务提议（proposal），并将该事务提议分发给所有的Follower服务器。
- Leader服务器需要等待所有Follower服务器的反馈。一旦超过半数的Follower服务器进行了正确的反馈后，那么Leader服务器将向**所有的**Follower服务器发送commit消息，要求将前一个事务提议刷新到磁盘。



ZAB协议包括2个基本的模式：消息广播 和 崩溃恢复







## 消息广播

ZAB的消息广播过程使用的是原子广播协议，类似于二阶段提交。针对客户端的请求，Leader服务器生成对应的事务提议，并将其发送给集群中所有的Follower服务器。然后收集各自的选票，最后进行事务提交。如图：

![image-20191019134027244](https://tva1.sinaimg.cn/large/006y8mN6gy1g83gfickamj318y0fsn31.jpg)

在ZAB协议中的二阶段提交，移除了中断逻辑。所有的Follower服务器要么正常反馈Leader提出的事务提议，要么就抛弃Leader服务器。同时，我们可以在过半的Follower服务器已经反馈ACK后，就开始提交事务提议了。

Leader服务器会为事务提议分配一个全局单调递增的ID，称为事务ID（ZXID）。由于ZAB协议需要保证每一个消息严格的因果关系，因此需要**将每一个事务提议按照其ZXID的先后顺序进行处理。**

在消息广播过程中，Leader服务器会为每一个Follower服务器分配一个队列，然后将事务提议依次放入到这些队列中去，并且根据FIFO的策略进行消息发送。

每一个Follower服务器接收到这个事务提议后，会把该事务提议**以事务日志的形式写入到本地磁盘中**，并且写入成功后，反馈给Leader服务器ACK。

当Leader服务器收到过半Follower服务器的ACK，就发送一个COMMIT消息，同时Leader自身完成事务提交，Follower服务器接收到COMMIT消息后，也进行事务提交。

之所以采用**原子广播协议**协议，是为了保证分布式数据一致性。过半的节点数据保存一致性。









## 崩溃恢复



### Leader选举

进入技术内幕——》Leader选举



### 数据同步

完成Leader选举之后，在正式开始工作之前，Leader服务器会去确认事务日志中所有事务提议（指已经提交的事务提议）是否都已经被过半的机器提交了，即是否完成数据同步。下面是ZAB协议的 数据同步过程。

Leader服务器为每一个Follower服务器准备一个队列，将那些没有被Follower服务器同步的事务以事务提议的形式逐个发送给Follower服务器，并在每一个事务提议消息后面发送一个commit消息，表示该事务已被提交。

等到Follower服务器将所有其未同步的事务提议都从Leader服务器上面同步过来，并且应用到本地数据库后，Leader服务器就会将该Follower服务器加入到真正可用的Follower列表中。



### ZXID的设计

ZXID是一个64位的数字。

其中低32位是一个简单的单调递增的计数器，Leader服务器产生一个新的事务提议的时候，都会对该计数器+1。

高32位，用来区分不同的Leader服务器。具体做法是，每选举产生一个新的Leader服务器，就会从Leader服务器的本地日志中取出一个最大的ZXID，生成对应的epoch值，然后再进行加1操作，之后就会以该值作为新的epoch。并将低32位从0开始生成ZXID。（我理解这里的epoch代表的就是一个Leader服务器的标志，每次选举Leader服务器，那么epoch值就会更新，代表是这段时期由这个新的Leader服务器进行事务请求的处理）。

ZAB协议中通过epoch编号来区分Leader周期变化，能够有效避免不同Leader服务器使用相同的ZXID。





## Paxos算法

Paxos算法是基于**消息传递**且具有**高度容错特性**的**一致性算法**，是目前公认的解决**分布式一致性**问题**最有效**的算法之一。



### 背景

在常见的分布式系统中，总会发生诸如**机器宕机**或**网络异常**（包括消息的延迟、丢失、重复、乱序，还有网络分区）等情况。Paxos算法需要解决的问题就是如何在一个可能发生上述异常的分布式系统中，快速且正确地在集群内部对**某个数据的值**达成**一致**，并且保证不论发生以上任何异常，都不会破坏整个系统的一致性。



### 算法陈述

Paxos算法有3个角色：

- Proposer：提案者（Leader）
- Acceptor：接收者（Follower），负责通过提案
- Learner：学习者，负责同步被通过的提案



分为2阶段

#### 阶段一

- Proposer选择一个提案编号，然后向所有的Acceptor发送编号为M的Prepare请求
- 如果一个Acceptor收到编号为M的prepare请求，且编号M大于该Acceptor已响应请求的所有编号，那么它就会将它已经批准过的最大编号的提案作为响应反馈给Propoer，同时该Acceptor会承若不会再批准任何编号小于M的提案。
  - ![image-20191207174956258](https://tva1.sinaimg.cn/large/006tNbRwgy1g9ob08lgxsj314k058tcr.jpg)



#### 阶段二

- 如果Propoer收到半数以上Acceptor对于编号M的prepare请求的响应，那么它就会发送一个[M, V]提案的Accept请求给Acceptor。
  - V的值就是收到的响应中提案最大的值。如果响应中不包含任何提案编号，那么它就是默认值
- 如果Acceptor接收到[M, V]提案的Accept请求，只要该Acceptor尚未对编号大于M的Prepare请求作出响应，它就可以通过这个提案









## ZAB与Paxos

ZAB协议并不是Paxos算法的一个典型实现，在讲解ZAB和Paxos的区别之前，我们先来看看两者的联系：

- 两者都存在一个类似Leader进程的角色，由其负责多个Follower进程的运行
- Leader进程都会等待超过半数的Follower做出正确的反馈后，才会将一个提案进行提交
- 在ZAB协议中，每个proposal都包含一个epoch值，用来代表当前的Leader周期。在Paxos算法中，同样存在这样的一个标识，只是名字变成了Ballot。



### 不同点

在Paxos的基础上，**ZAB协议额外增加了一个同步阶段。** 在选举结束后，进入同步阶段，新的Leader会确保存过半的Follower已经提交了上一个Leader周期中的所有事务提议。

