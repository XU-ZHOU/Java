**Kafka知识总结**

**1.讲讲acks参数对消息持久化的影响**

**目录**

1. 写在前面
2. 如何保证宕机时数据不丢失？
3. 多副本之间数据如何同步？
4. ISR到底指的是什么东西？
5. acks参数的含义？
6. 最后的思考

**1.写在前面**

面试大厂时，一旦简历上写了Kafka，几乎必然会被问到一个问题：说说acks参数对消息持久化的影响？

这个acks参数在kafka的使用中，是非常核心以及关键的一个参数，决定了很多东西。

所以无论是为了面试还是实际项目使用，大家都值得看一下这篇文章对Kafka的acks参数的分析，以及背后的原理。

**2.如何保证宕机的时候数据不丢失？（或者kafka如何保证高可用、或者Kafka如何保证高可用）**

- Kafka 一个最基本的架构认识：由多个 broker 组成，每个 broker 是一个节点；创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。

  这就是**天然的分布式消息队列**，就是说一个 topic 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**。

- 而且Kafka还提供replica**副本机制**，每个partition的数据都会同步到其他机器上，形成自己的多个replica副本。所有replica会选举出来一个leader出来，那么**生产和消费都跟这个leader打交道**，然后其他replica就是follower。写的时候，leader会负责把数据同步到所有follower上去，读的时候就直接读leader上的数据即可。

如果某个broker宕机了，那个broker上的partition在其他机器上都有副本。如果这个宕机的broker上面有某个partition的leader，那么从follower中重新选举一个新的leader出来，然后继续读写新的leader即可，这就是所谓的高可用。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/1.jpg>)

3.**多副本之间数据如何保证同步**

其实任何一个Partition，只有Leader是对外提供读写服务的，也就是说，如果有一个客户端往一个Partition写入数据，此时一般就是写入这个Partition的Leader副本。

然后Leader副本接收到数据之后，Follower副本会不停的给他发送请求尝试去拉取最新的数据，拉取到自己本地后，写入磁盘中。如下图所示：

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/2.jpg>)

**4.ISR到底指的是什么东西？**

ISR全称是“In-Sync Replicas”，也就是**保持同步的副本**，他的含义就是，跟Leader始终保持同步的Follower有哪些。

大家可以想一下 ，如果说某个Follower所在的Broker因为JVM FullGC之类的问题，导致自己卡顿了，无法及时从Leader拉取同步数据，那么是不是会导致Follower的数据比Leader要落后很多？

所以这个时候，就意味着Follower已经跟Leader不再处于同步的关系了。但是只要Follower一直及时从Leader同步数据，就可以保证他们是处于同步的关系的。

所以每个Partition都有一个ISR，这个ISR里一定会有Leader自己，因为Leader肯定数据是最新的，然后就是那些跟Leader保持同步的Follower，也会在ISR里。

**5.acks参数的含义**

首先这个acks参数，是在KafkaProducer，也就是生产者客户端里设置的

也就是说，你往kafka写数据的时候，就可以来设置这个acks参数。然后这个参数实际上有三种常见的值可以设置，分别是：**0、1 和 all**。

**第一种选择是把acks参数设置为0**，意思就是我的KafkaProducer在客户端，只要把消息发送出去，不管那条数据有没有在哪怕Partition Leader上落到磁盘，我就不管他了，直接就认为这个消息发送成功了。

如果你采用这种设置的话，那么你必须注意的一点是，可能你发送出去的消息还在半路。结果呢，Partition Leader所在Broker就直接挂了，然后结果你的客户端还认为消息发送成功了，此时就会**导致这条消息就丢失了**。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/3.jpg>)

**第二种选择是设置 acks = 1**，意思就是说只要Partition Leader接收到消息而且写入本地磁盘了，就认为成功了，不管他其他的Follower有没有同步过去这条消息了。

这种设置其实是**kafka默认的设置**

也就是说，默认情况下，你要是不管acks这个参数，只要Partition Leader写成功就算成功。

但是这里有一个问题，万一Partition Leader刚刚接收到消息，Follower还没来得及同步过去，结果Leader所在的broker宕机了，此时也会导致这条消息丢失，因为人家客户端已经认为发送成功了。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/4.jpg>)

**最后一种情况，就是设置acks=all**，这个意思就是说，**Partition Leader接收到消息之后，还必须要求ISR列表里跟Leader保持同步的那些Follower都要把消息同步过去**，才能认为这条消息是写入成功了。

如果说Partition Leader刚接收到了消息，但是结果Follower没有收到消息，此时Leader宕机了，那么客户端会感知到这个消息没发送成功，他会重试再次发送消息过去。

此时可能Partition 2的Follower变成Leader了，此时ISR列表里只有最新的这个Follower转变成的Leader了，那么只要这个新的Leader接收消息就算成功了。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/5.jpg>)

**6.最后的思考**

acks=all 就可以代表数据一定不会丢失了吗？

当然不是，如果你的Partition只有一个副本，也就是一个Leader，任何Follower都没有，你认为acks=all有用吗？

当然没用了，因为ISR里就一个Leader，他接收完消息后宕机，也会导致数据丢失。

所以说，**这个acks=all，必须跟ISR列表里至少有2个以上的副本配合使用**，起码是有一个Leader和一个Follower才可以。

这样才能保证说写一条数据过去，一定是2个以上的副本都收到了才算是成功，此时任何一个副本宕机，不会导致数据丢失。

参考：https://mp.weixin.qq.com/s/IxS46JAr7D9sBtCDr8pd7A