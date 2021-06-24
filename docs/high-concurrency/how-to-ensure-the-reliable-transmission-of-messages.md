## 面试题
如何保证消息的可靠性传输？或者说，如何处理消息丢失的问题？

## 面试官心理分析
这个是肯定的，用 MQ 有个基本原则，就是**数据不能多一条，也不能少一条**，不能多，就是前面说的[重复消费和幂等性问题](/docs/high-concurrency/how-to-ensure-that-messages-are-not-repeatedly-consumed.md)。不能少，就是说这数据别搞丢了。那这个问题你必须得考虑一下。

如果说你这个是用 MQ 来传递非常核心的消息，比如说计费、扣费的一些消息，那必须确保这个 MQ 传递过程中**绝对不会把计费消息给弄丢**。

## 面试题剖析
数据的丢失问题，可能出现在生产者、MQ、消费者中，咱们从 RabbitMQ 和 Kafka 分别来分析一下吧。

### RabbitMQ
![rabbitmq-message-lose](/images/rabbitmq-message-lose.png)

#### 生产者弄丢了数据

生产者将数据发送到 RabbitMQ 的时候，可能数据就在半路给搞丢了，因为网络问题啥的，都有可能。

此时可以选择用 RabbitMQ 提供的事务功能，就是生产者**发送数据之前**开启 RabbitMQ 事务`channel.txSelect`，然后发送消息，如果消息没有成功被 RabbitMQ 接收到，那么生产者会收到异常报错，此时就可以回滚事务`channel.txRollback`，然后重试发送消息；如果收到了消息，那么可以提交事务`channel.txCommit`。
```java
// 开启事务
channel.txSelect
try {
    // 这里发送消息
} catch (Exception e) {
    channel.txRollback

    // 这里再次重发这条消息
}

// 提交事务
channel.txCommit
```

但是问题是，RabbitMQ 事务机制（同步）一搞，基本上**吞吐量会下来，因为太耗性能**。

所以一般来说，如果你要确保说写 RabbitMQ 的消息别丢，可以开启 `confirm` 模式，在生产者那里设置开启 `confirm` 模式之后，你每次写的消息都会分配一个唯一的 id，然后如果写入了 RabbitMQ 中，RabbitMQ 会给你回传一个 `ack` 消息，告诉你说这个消息 ok 了。如果 RabbitMQ 没能处理这个消息，会回调你的一个 `nack` 接口，告诉你这个消息接收失败，你可以重试。而且你可以结合这个机制自己在内存里维护每个消息 id 的状态，如果超过一定时间还没接收到这个消息的回调，那么你可以重发。

事务机制和 `confirm` 机制最大的不同在于，**事务机制是同步的**，你提交一个事务之后会**阻塞**在那儿，但是 `confirm` 机制是**异步**的，你发送个消息之后就可以发送下一个消息，然后那个消息 RabbitMQ 接收了之后会异步回调你的一个接口通知你这个消息接收到了。

所以一般在生产者这块**避免数据丢失**，都是用 `confirm` 机制的。

#### RabbitMQ 弄丢了数据
就是 RabbitMQ 自己弄丢了数据，这个你必须**开启 RabbitMQ 的持久化**，就是消息写入之后会持久化到磁盘，哪怕是 RabbitMQ 自己挂了，**恢复之后会自动读取之前存储的数据**，一般数据不会丢。除非极其罕见的是，RabbitMQ 还没持久化，自己就挂了，**可能导致少量数据丢失**，但是这个概率较小。

设置持久化有**两个步骤**：

- 创建 queue 的时候将其设置为持久化<br>
这样就可以保证 RabbitMQ 持久化 queue 的元数据，但是它是不会持久化 queue 里的数据的。
- 第二个是发送消息的时候将消息的 `deliveryMode` 设置为 2<br>
就是将消息设置为持久化的，此时 RabbitMQ 就会将消息持久化到磁盘上去。

必须要同时设置这两个持久化才行，RabbitMQ 哪怕是挂了，再次重启，也会从磁盘上重启恢复 queue，恢复这个 queue 里的数据。

注意，哪怕是你给 RabbitMQ 开启了持久化机制，也有一种可能，就是这个消息写到了 RabbitMQ 中，但是还没来得及持久化到磁盘上，结果不巧，此时 RabbitMQ 挂了，就会导致内存里的一点点数据丢失。

所以，持久化可以跟生产者那边的 `confirm` 机制配合起来，只有消息被持久化到磁盘之后，才会通知生产者 `ack` 了，所以哪怕是在持久化到磁盘之前，RabbitMQ 挂了，数据丢了，生产者收不到 `ack`，你也是可以自己重发的。

#### 消费端弄丢了数据
RabbitMQ 如果丢失了数据，主要是因为你消费的时候，**刚消费到，还没处理，结果进程挂了**，比如重启了，那么就尴尬了，RabbitMQ 认为你都消费了，这数据就丢了。

这个时候得用 RabbitMQ 提供的 `ack` 机制，简单来说，就是你必须关闭 RabbitMQ 的自动 `ack`，可以通过一个 api 来调用就行，然后每次你自己代码里确保处理完的时候，再在程序里 `ack` 一把。这样的话，如果你还没处理完，不就没有 `ack` 了？那 RabbitMQ 就认为你还没处理完，这个时候 RabbitMQ 会把这个消费分配给别的 consumer 去处理，消息是不会丢的。

![rabbitmq-message-lose-solution](/images/rabbitmq-message-lose-solution.png)

### Kafka

#### 消费端弄丢了数据
唯一可能导致消费者弄丢数据的情况，就是说，你消费到了这个消息，然后消费者那边**自动提交了 offset**，让 Kafka 以为你已经消费好了这个消息，但其实你才刚准备处理这个消息，你还没处理，你自己就挂了，此时这条消息就丢咯。

这不是跟 RabbitMQ 差不多吗，大家都知道 Kafka 会自动提交 offset，那么只要**关闭自动提交** offset，在处理完之后自己手动提交 offset，就可以保证数据不会丢。但是此时确实还是**可能会有重复消费**，比如你刚处理完，还没提交 offset，结果自己挂了，此时肯定会重复消费一次，自己保证幂等性就好了。

生产环境碰到的一个问题，就是说我们的 Kafka 消费者消费到了数据之后是写到一个内存的 queue 里先缓冲一下，结果有的时候，你刚把消息写入内存 queue，然后消费者会自动提交 offset。然后此时我们重启了系统，就会导致内存 queue 里还没来得及处理的数据就丢失了。

#### Kafka 弄丢了数据

这块比较常见的一个场景，就是 Kafka 某个 broker 宕机，然后重新选举 partition 的 leader。大家想想，要是此时其他的 follower 刚好还有些数据没有同步，结果此时 leader 挂了，然后选举某个 follower 成 leader 之后，不就少了一些数据？这就丢了一些数据啊。

生产环境也遇到过，我们也是，之前 Kafka 的 leader 机器宕机了，将 follower 切换为 leader 之后，就会发现说这个数据就丢了。

所以此时一般是要求起码设置如下 4 个参数：

- 给 topic 设置 `replication.factor` 参数：这个值必须大于 1，要求每个 partition 必须有至少 2 个副本。
- 在 Kafka 服务端设置 `min.insync.replicas` 参数：这个值必须大于 1，这个是要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队，这样才能确保 leader 挂了还有一个 follower 吧。
- 在 producer 端设置 `acks=all`：这个是要求每条数据，必须是**写入所有 replica 之后，才能认为是写成功了**。
- 在 producer 端设置 `retries=MAX`（很大很大很大的一个值，无限次重试的意思）：这个是**要求一旦写入失败，就无限重试**，卡在这里了。

我们生产环境就是按照上述要求配置的，这样配置之后，至少在 Kafka broker 端就可以保证在 leader 所在 broker 发生故障，进行 leader 切换时，数据不会丢失。

#### 生产者会不会弄丢数据？
如果按照上述的思路设置了 `acks=all`，一定不会丢，要求是，你的 leader 接收到消息，所有的 follower 都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试，重试无限次。









Rabbitmq专题：rabbitMQ如何保证消息的可靠性投递？如何防止消息丢失
![图片](https://user-images.githubusercontent.com/65522342/123244374-7393e080-d516-11eb-9e3a-71d97ce3ddbf.png)

    1. 消息可能出现丢失的情况
    2. 生产者如何保证消息的可靠性投递
        2.1 消息落库打标 + confirm机制
        2.2 消息幂等性如何保证？
        2.3 延时消息确认
    3. rabbitMQ服务器如何防止消息丢失
    4. 消费者如何防止消息丢失


1. 消息可能出现丢失的情况

在这里插入图片描述
消息可能出现丢失的情况如上图所示，针对生产者、MQ、消费者三个维度都可能出现消息丢失

    生产者在向MQ服务器Broker发送message时，可能由于网络原因，消息发送失败，在传输过程中丢失，此时消息还未到达MQ服务器

    RabbitMq服务器接收到消息，此时RabbitMq服务器突然宕机，造成消息丢失

    消费端拿到消息后，还未来得及处理就宕机或者被重启了，造成消息丢失


2. 生产者如何保证消息的可靠性投递
2.1 消息落库打标 + confirm机制

        针对生产者在向Broker发送message时的消息丢失，可以使用消息入库打标记并配合mq的confirm机制来保证消息可靠性，首先要了解什么是mq 的confirm机制？生产者向mq服务端发送消息时，mq服务器会根据消息的接收情况给生产者一个应答，生产者根据应答情况来确保该条消息是否成功的发送到了mq服务器！而这个应答的过程就是mq的confirm机制。

confirm机制的现实步骤如下：

    在生产者的channel 上开启confirm机制channel.confirmSelect();
    在生产者的channel上添加监听，用来监听mq-server返回的应答

伪代码如下：

     //开启confirm机制
     channel.confirmSelect();
     
     //设置confirm 监听
     channel.addConfirmListener(new AngleConfirmListerner());
     
     //......发送消息 
     //注意：生产者连接不能断开，否则无法监听回调

================= 消息监听器AngleConfirmListerner =================

	public class AngleConfirmListerner implements ConfirmListener {
	
		//broker正常签收
	    @Override
	    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
	        System.out.println("消息deliveryTag" + deliveryTag + "被正常签收");
	    }
	
		//broker异常签收
	    @Override
	    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
	        System.out.println("消息deliveryTag" + deliveryTag + "没被签收");
	    }
	 }


消息入库打标解决思路

以订单系统向mq发消息为例
![图片](https://user-images.githubusercontent.com/65522342/123244468-8ad2ce00-d516-11eb-89cc-46c4e830a454.png)

发消息链路流程

    把订单数据入库，同时构建消息数据，消息数据同样入库，记录在message表中，消息状态初始标记为0
    生产者(订单服务)发送消息到mq服务器
    mq会通过confirm机制返回确认消息，订单服务监听回调的confirm结果
    正常情况下，mq接收消息成功并返回ACK，订单服务监听到此ACK，并更新message表中的消息状态标记为1，表示发送与接收正常！
    异常情况下，由于网络闪断，导致消费端监控mq服务访问的确认消息 没有收到，那么在msg_db中的那条消息的 状态永远就是0状态。这个时候，我们需要对这种情况下做出补偿
    根据消费者(库存服务)消费情况（消费正常、消费异常），分别修改message表中的消息状态为3或4！

异常情况下的补偿机制

        启动一个定时任务，去扫描message这个消息记录表，针对消息状态为0的消息，根据业务来设置重发规则

    ①：插入message表中的消息，如果3分钟后状态还是0，代表未发送或发送失败，那么进行消息重发，并记录消息重发次数
    ②：如果消息重发次数大于5次，且消息状态还是0的时候，就把这条消息状态设置为2，代表消息发送不成功，此时人工介入，调查未成功原因！


2.2 消息幂等性如何保证？

        幂等性简而言之，就是对接口发起的一次调用和多次调用，所产生的结果都是一致的。某些接口具有天然的幂等性: 比如查询接口，不管是查询一次还是多次，返回的结果都是一致的。

        但对于标题2.1的发消息链路中，mq返回成功的ACK时，如果因为网络原因ACK发送失败，就导致消息生产者(订单服务)无法修改message表中的消息状态，又因为补偿机制的存在，回轮询扫描、判断并重发，如果没有幂等性保证，就会造成订单的重复提交！ 再或者用户多次点击提交订单，如果没有接口幂等性，也会造成订单重复提交！

        
订单重复提交幂等性解决方案

        如果使用户由于网络卡顿而心急，不断点击提交订单造成的订单重复提交，可以为订单生成一个全局唯一性ID(订单号+业务类型)，并把该唯一id使用setnx命令保存在redis，在第一次保存的时候，由于redis中没有该key,那么就会把全局唯一ID 设置上，此时订单入库保存。若出现前端重复点击按钮, 由于第一步已经setnx上了 ，就会阻止后面的保存

        
mq服务端是如何保证幂等性的？

        mq服务端在接受消息时，会对每一条消息都生成一个全局唯一的与业务无关的ID(inner_msg_id)，先根据inner_msg_id 是否需要重复发送，再决定消息是否落地 。这样保证每条消息都只会在mq服务端落地一次。如果由于网络原因mq落地成功，但返回ack失败，生产者由于补偿机制重发消息，mq服务端会对比新消息的inner_msg_id，由于此条消息在mq服务器已落地，所以id相等，不予处理！


2.3 延时消息确认

        消息入库打标存在缺点：在消息入库打标第一步的过程中，既插入了业务数据表，也同时插入了消息记录表，进行了二次db操作，在分布式环境下，可能还要保证分布式事务。延时消息确认机制相比消息入库打标，减少了一次message入库操作，不用加分布式事务，系统速度显著提高。延时消息的思路如下：

![图片](https://user-images.githubusercontent.com/65522342/123244564-a047f800-d516-11eb-888e-cb19a8956201.png)
步骤如下：

    订单服务首先将业务代码入库，注意：消息并没有入库
    发送业务消息给mq
    发送第二个延迟确认消息，与业务消息发送时间有一定间隔（1分钟），保证消费端处理完毕并消息入库之后才发送！
    库存服务监听第二步发送的业务消息进行消费
    消费端（库存服务）发送确认消息ack到mq，此时第三步还没有执行，间隔时间未到！
    回调服务监听到这个确认消息
    把这个消费端完成消费的确认消息入库
    回调服务检查到延迟确认消息，会在数据库查询是否有这条消息
    如果没有查到这条消息，说明第五步库存服务消费消息失败了。此时回调服务通过RPC给一个重新发送命令到上游系统


3. rabbitMQ服务器如何防止消息丢失

        RabbitMQ 的消息默认存放在内存上面，如果不特别声明设置，消息不会持久化保存到硬盘上面的，如果节点重启或者意外crash掉，消息就会丢失。所以就要对消息进行持久化处理。如何持久化？要想做到消息持久化，必须满足以下三个条件，缺一不可。下面具体说明下：

    Exchange 设置持久化

/**
 * exchangeName：交换机名称
 * exchangeType：交换机类型
 * true: 开启消息持久化
 * false：代表连接停掉后不自动删除掉
 * null：其他参数
 */
channel.exchangeDeclare(exchangeName,exchangeType,true,false,null);

    Queue 设置持久化

/**
* queueName：队列名称
* true: 开启消息持久化
* false：代表所有消费者都可以访问，true代表只有第一次拥有它的消费者才能一直使用，其他消费者不让访问
* false：代表连接停掉后不自动删除掉
* null：其他参数
*/
channel.queueDeclare(queueName,true,false,false,null);
Message持久化发送：发送消息设置发送模式deliveryMode=2，代表持久化消息

//消息属性
AMQP.BasicProperties basicProperties = new AMQP.BasicProperties().builder()
        .deliveryMode(2)//消息持久化
        .contentEncoding("UTF-8")
        .correlationId(UUID.randomUUID().toString())
        .headers(infoMap)
        .build();
 
//生产者发送消息
channel.basicPublish(exchangeName,routingKey,basicProperties,(msgBody+i).getBytes());

4. 消费者如何防止消息丢失

首先看一下，生产者、mq服务器、消费者之间的消息流转过程
![图片](https://user-images.githubusercontent.com/65522342/123244610-accc5080-d516-11eb-8648-a1e028aa0bbd.png)

    第一步：消息生产者向Mq服务端发送消息
    第二步：mq 服务端把消息进行落地
    第三步：mq 服务端向消息生产者发送ack
    第四步：消息消费者从mq服务端拉取消息消费
    第五步：消费者向mq服务端发送ack
    第六步：mq服务端将落地消息删除

        第四步消费者获取到消息之后，没有来得及处理完毕，自己直接宕机了,因为消息者默认采用的是自动ack，此时RabbitMQ的自动ack机制会通知MQ Server这条消息已经处理好了，此时消息就丢了，并不是预期的。

        那么我们可以采用手动ack机制来解决这个问题，消费端处理完逻辑之后再通知MQ Server，这样消费者没处理完消息不会发送ack,如果在消费者拿到消息，没来得及处理的情况下自己挂了，此时MQ集群会自动感知到，它就会自觉的重发消息给其他的消费者服务实例。

根据上面的思路你需要完成下面的两步操作：

①：消费者关闭自动ack

 //第二个参数为false代表不自动ack
 channel.basicConsume(queueName,false,new TulingAckConsumer(channel));

    1
    2

②：消费完成才ack，否则不ack

  try{
      //模拟业务
      Integer mark = (Integer) properties.getHeaders().get("mark");
      if(mark != 0 ) {
          //模拟消息消费
          System.out.println("消费消息:"+new String(body));
          //消费完成，发出ack通知
          channel.basicAck(envelope.getDeliveryTag(),false);
      }else{
          //否则抛异常
          throw new RuntimeException("模拟业务异常");
      }
  }catch (Exception e) {
      System.out.println("异常消费消息:"+new String(body));
      
      //捕捉异常，消息重回队列，让其他集群节点消费
      //channel.basicNack(envelope.getDeliveryTag(),false,true);
      
      //不重回队列，杀死这个消息
      channel.basicNack(envelope.getDeliveryTag(),false,false);

  }
————————————————

