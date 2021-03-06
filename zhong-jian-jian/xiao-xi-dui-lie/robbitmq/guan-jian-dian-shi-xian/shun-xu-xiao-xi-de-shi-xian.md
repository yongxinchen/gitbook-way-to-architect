# 顺序消息的实现

## 什么是顺序消息

消息有序指的是可以按照消息的发送顺序来消费。例如：一笔订单产生了 3 条消息，分别是订单创建、订单付款、订单完成。消费时，要按照顺序依次消费才有意义。与此同时多笔订单之间又是可以并行消费的（即：**全局无序，局部有序**），如下图所示：

![](../../../../.gitbook/assets/image%20%2810%29.png)

> 全局有序和局部有序
>
> 这里的全局和局部，是针对消息而言的，全局有序即全部的消息都要有序，局部有序是指所有的消息无序，但是同一属性的消息（比如同一订单下的消息）是有序的。

**如何保证顺序**

在MQ的模型中，顺序需要由3个阶段去保障：

1. 消息被发送时保持顺序
2. 消息被存储时保持和发送的顺序一致
3. 消息被消费时保持和存储的顺序一致

发送时保持顺序意味着对于有顺序要求的消息，用户应该在同一个线程中采用同步的方式发送。存储保持和发送的顺序一致则要求在同一线程中被发送出来的消息A和B，存储时在空间上A一定在B之前。而消费保持和存储一致则要求消息A、B到达Consumer之后必须按照先A后B的顺序被处理，如下图所示：

![](https://images2018.cnblogs.com/blog/471426/201805/471426-20180519131211273-554395305.png)

对于两个订单的消息的原始数据：a1、b1、b2、a2、a3、b3（绝对时间下发生的顺序）：

* 在发送时，a订单的消息需要保持a1、a2、a3的顺序，b订单的消息也相同，但是a、b订单之间的消息没有顺序关系，这意味着a、b订单的消息可以在不同的线程中被发送出去
* 在存储时，需要分别保证a、b订单的消息的顺序，但是a、b订单之间的消息的顺序可以不保证
  * a1、b1、b2、a2、a3、b3是可以接受的
  * a1、a2、b1、b2、a3、b3也是可以接受的
  * a1、a3、b1、b2、a2、b3是不能接受的
* 消费时保证顺序的简单方式就是“什么都不做”，不对收到的消息的顺序进行调整，即只要一个分区的消息只由一个线程处理即可；当然，如果a、b在一个分区中，在收到消息后也可以将他们拆分到不同线程中处理，不过要权衡一下收益。

## **RocketMQ中顺序消息的实现**

### **Producer端**

`Producer`端确保消息顺序唯一要做的事情就是将消息路由到特定的分区，在RocketMQ中，通过`MessageQueueSelector`来实现分区的选择。

```java
// RocketMQ通过MessageQueueSelector中实现的算法来确定消息发送到哪一个队列上
// RocketMQ默认提供了两种MessageQueueSelector实现：随机和Hash
// 可以根据业务实现自己的MessageQueueSelector来决定消息按照何种策略发送到消息队列中
SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
    }
}, orderId);
```

在获取到路由信息以后，会根据`MessageQueueSelector`实现的算法来选择一个队列，同一个`orderId`获取到的肯定是同一个队列。

这里，其实隐含了一个信息：**消息是同步发送的**，上述send方法，最终会调用如下的方法：

```java
public SendResult send(Message msg, MessageQueueSelector selector, Object arg, long timeout)
        throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendSelectImpl(msg, selector, arg, CommunicationMode.SYNC, null, timeout);
}
```

 其中，CommunicationMode.SYNC表示进行同步通讯。同步发送表示，producer发送消息之后不会立即返回，会等待broker的response。producer在收到broker的response并且是处理成功之后才算是消息发送成功。

也就是说，broker不负责对消息的顺序进行处理，只会按照接收到的消息的顺序进行存储。

### **Consumer端**

RocketMQ消费端有两种类型：MQPullConsumer和MQPushConsumer。

#### MQPullConsumer

MQPullConsumer由用户控制线程，主动从服务端获取消息，每次获取到的是一个MessageQueue中的消息。PullResult中的List msgFoundList自然和存储顺序一致，用户需要再拿到这批消息后自己保证消费的顺序。

#### MQPushConsumer

对于PushConsumer，由用户注册MessageListener来消费消息。顺序消费和普通消费的listener是不一样的，顺序消费需要实现的是下面这个接口

`org.apache.rocketmq.client.consumer.listener.MessageListenerOrderly`

在consumer的`start()`方法中，会根据`listener`的类型判断应该使用哪一个service来消费：

```java
if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
    this.consumeOrderly = true;
    this.consumeMessageService =
        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
} else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
    this.consumeOrderly = false;
    this.consumeMessageService =
        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
}
```

consumer拉取消息是按照offset拉取的，所以consumer能保证拉取到consumer的消息是连续有序的，在拉取消息后，将消息放到`ProcessQueue`这个队列中：

```java
//pullResult.getMsgFoundList()中的消息是拉取来的消息，是有序的，然后将其放到processQueue中，
boolean dispathToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
```

一个MessageQueue对应一个ProcessQueue，ProcessQueue是一个有序队列，该队列记录一个queueId下所有从broker pull回来的消息，如果消费成功了就会从队列中删除。ProcessQueue有序的原因是维护了一个TreeMap：

```java
//key是当前消息的offset, 所以msgTreeMap里面的消息是按照offset排序的。
private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<Long, MessageExt>();
```

 之后，启动线程执行`ConsumeRequest.run`方法来消费`ProcessQueue`中的消息。在run\(\)方法中，会首先获取messageQueue的锁，保证一个messageQueue只有一个线程访问，因为顺序消息只有一个队列，也就保证了只有一个线程消费。

```java
final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
```

> 那为什么还要线程池：因为会有不同多个messageQueue，同一个messageQueue中的消息是顺序的，由同一线程进行消费，多个线程则可以同时对多个不同的messageQueue进行处理。

上述逻辑可以用下图来表示：

![](https://images2018.cnblogs.com/blog/471426/201805/471426-20180519131253178-1043342972.png)

## 总结

1、全局有序即全部的消息都要有序，局部有序是指所有的消息无序，但是同一属性的消息（比如同一订单下的消息）是有序的。

2、RocketMQ中的实现是靠Producer和Consumer，Broker只负责按接收到消息的顺序进行存储。

3、Producer使用同步发送的方式，通过自定义MessageQueueSelector将想要有序的消息发送到同一队列中

4、Consumer中，MQPullConsumer比较简单，因为是客户端主动拉取消息，只要按照拉取的消息顺序进行消费即可（用户自行控制）；MQPushConsumer会从Broker按顺序获取消息，且是按照offset顺序获取，获取后放到内部结构为TreeMap&lt;queueId，message&gt;的ProcessQueue中，保证消息ProcessQueue中的顺序，然后交给线程池处理，处理时通过`synchronize(MessageQueue)`的方式保证同一队列中的消息只会交给同一个线程进行处理。

5、在实践中，尽量避免顺序消费，因为顺序消费的成本较高，上述RocketMQ实现顺序消费，其实有一个前提：Producer每条消息都会发送成功！但是Producer并不保证消息必达，发送失败重试n次后，依然会丢弃，仍需要用户自行处理这种消息发送失败的情形。假如发送顺序消息a1，a2，a3，三者均为同步发送，假如a1发送失败（Producer内部重试N次后丢弃消息的情况），a2和a3发送成功，仍会给业务造成问题。可以通过业务逻辑来避免消息无序造成的问题：比如上面的订单消息：订单创建、订单付款、订单完成，假如首先接收到了订单完成的消息，可以通过查询订单是否创建，是否付款来决定是否处理这条消息或者将反馈给Broker消息消费失败。



## 参考

[分布式开放消息系统\(RocketMQ\)的原理与实践](https://www.jianshu.com/p/453c6e7ff81c)

[RocketMQ源码 — 十、 RocketMQ顺序消息](https://blog.csdn.net/gesanghuakaisunshine/article/details/80414029)

[聊一聊顺序消息（RocketMQ顺序消息的实现机制）](https://www.cnblogs.com/hzmark/p/orderly_message.html)

[RocketMQ源码学习之三-异步消息发送](http://xiajunhust.github.io/2016/11/14/RocketMQ%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E4%B9%8B%E4%B8%89-%E5%BC%82%E6%AD%A5%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81/)

