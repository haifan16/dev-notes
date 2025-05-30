## 消息的特殊处理

### 错乱的消息顺序

消息有序指的是可以按照消息的发送顺序来消费(FIFO)。RocketMQ 可以严格地保证消息有序，可以分为*分区有序*或者*全局有序*。

顺序消费的原理解析：

在默认情况下，消息发送会采取 Round Robin 轮询方式把消息发送到不同的 queue(分区队列)；而消费消息的时候从多个 queue 上拉取消息，这种情况发送和消费是不能保证顺序的。

但是如果控制发送的顺序消息只依次发送到同一个 queue 中，消费的时候只从这个 queue 上依次拉取，就能够保证顺序。

当所有消息都路由到同一个 queue 且被顺序消费时，称为**全局有序**；若消息被分散到多个 queue，但每个 queue 内部保持顺序，则称为**分区有序**（或局部有序）。

以订单业务为例，一个订单的顺序流程是：创建、付款、推送、完成。订单号相同的消息会被先后发送到同一个队列中，消费时，同一个 OrderId 获取到的肯定是同一个队列。

~~（TODO：画图）~~

### 顺序消息

1. 订单步骤实体类

```Java
/**
 * 订单的步骤
 */
public class OrderStep {
	private long orderId;
	private String desc;
	
	public long getOrderId() {
		return orderId;
	}
	
	public void setOrderId(long orderId) {
		this.orderId = orderId;
	}
	
	public String getDesc() {
		return desc;
	}
	
	public void setDesc(String desc) {
		this.desc = desc;
	}
	
	@Override
	public String toString() {
		return "OrderStep{" +
				"orderId=" + orderId +
				", desc='" + desc + '\'' +
				'}';
	}
}
```

2. 发送消息

```Java
public class Producer {
	public static void main(String[] args) throws Exception {
		DefaultMQProducer producer = new DefaultMQProducer("group1");
		producer.setNamesrvAddr("localhost:9876");
		producer.start();
		
		List<OrderStep> orderList = new Producer.buildOrders();
		
		// 设置消息进入到指定的消息队列中
		for (final OrderStep order : orderList) {
			Message msg = new Message("topic1", order.toString().getBytes());
			// 发送时要指定对应的消息队列选择器
			producer.send(msg, new MessageQueueSelector() {
				// 设置当前消息发送时使用哪一个消息队列
				@Override
				public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
					// 根据发送的消息不同，选择不同的消息队列
					// 根据 id 来选择一个消息队列的对象，并返回 int 值
					Long orderId = order.getOrderId();
					long index = orderId % mqs.size();
					return mqs.get((int) index);
				}
			}, mull);
			System.out.println(result);
		}
		
		producer.shutdown();
	}
	
	/**
	 * 生成模拟订单数据
	 */
	private List<OrderStep> buildOrders() {
		List<OrderStep> orderList = new ArrayList<OrderStep>();
		
		// 1L 创建
		OrderStep orderDemo = new OrderStep();
		orderDemo.setOrderId(1L);
		orderDemo.setDesc("创建");
		orderList.add(orderDemo);
		// 2L 创建
		orderDemo = new OrderStep();
		orderDemo.setOrderId(2L);
		orderDemo.setDesc("创建");
		orderList.add(orderDemo);
		// 1L 付款 ...
		// 3L 创建 ...
		// 2L 付款 ...
		// 3L 付款 ...
		// 2L 完成 ...
		// 1L 推送 ...
		// 3L 完成 ...
		// 1L 完成 ...
		
		return orderList;
	}
}
```

3. 接收消息

```Java
// 使用单线程的模式从消息队列中取数据，一个线程绑定一个消息队列
consumer.registerMessageListener(new MessageListenerOrderly() {
    // 使用 MessageListenerOrderly 接口后，对消息队列的处理由【一个消息队列多个线程服务】转化为【一个消息队列一个线程服务】
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
        for (MessageExt msg : list) {
            System.out.println(Thread.currentThread().getName() + "。消息：" + new String(msg.getBody()) + "。queueId:" + msg getQueueId());
        }
    }
})
```

### 事务消息

#### 流程

![alt text](<RocketMQ 事务补偿过程-1.png>)

~~（TODO：重新画图）~~

1. 正常事务过程
2. 事务补偿过程

#### 三种事务消息状态

1. 提交状态：允许进入队列，此消息与非事务消息没区别
2. 回滚状态：不允许进入队列，此消息等同于未发送过
3. 中间状态：完成了 half 消息的发送，未对 MQ 进行二次状态确认

注意：事务消息仅与生产者有关，与消费者无关

#### 代码

1. 提交状态

```java
// 事务消息使用的生产者是 TransactionMQProducer
TransactionMQProducer producer = new TransactionMQProducer("group1");
producer.setNamesrvAddr("localhost:9876");
// 添加本地事务对应的监听
producer.setTransactionListener(new TransactionListener() {
    // 正常事务过程
        @Overrider
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        return LocalTransactionState.COMMIT_MESSAGE;
    }
    // 事务补偿过程
        @Overrider
    public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
        return null;
    }
});
producer.start();
Message msg = new Message("topic8", ("事务消息：Hello RocketMQ!").getBytes("UTF-8"));
SendResult result = producer.sendMessageInTransaction(msg, null);
System.out.println("发送结果：" + result);
producer.shutdown();
```

2. 回滚状态

```Java
// 添加本地事务对应的监听
producer.setTransactionListener(new TransactionListener() {
    // 正常事务过程
        @Overrider
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        return LocalTransactionState.ROLLBACK_MESSAGE; // 回滚状态
    }
    // 事务补偿过程
        @Overrider
    public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
        return null;
    }
});
```

3. 中间状态

```Java
public static void main(String[] args) throws Exception {
    TransactionMQProducer producer = new TransactionMQProducer("group1");
    producer.setNamesrvAddr("localhost:9876");
    producer.setTransactionListener(new TransactionListener() {
        // 正常事务过程
        @Overrider
        public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
            return LocalTransactionState.UNKNOW;
        }
        // 事务补偿过程（正常执行 UNKNOW 才会触发）
        @Override
        public LocalTransactionState checkLocalTransaction(MessageExt msg) {
            System.out.println("事务补偿过程");
            return LocalTransactionState.COMMIT_MESSAGE;
        }
    });
    producer.start();
    Message msg = new Message("topic13", "Hello RocketMQ!".getBytes("UTF-8"));
    SendResult result = producer.sendMessageInTransaction(msg, null);
    System.out.println("返回结果：" + result);

    // 注意：事务补偿，生产者要一直启动着！
}
```

## 集群搭建

### RocketMQ 集群分类

1. 单机
    1. 一个 broker 提供服务（宕机后服务瘫痪）
2. 集群
    1. 多个 broker 提供服务（单机宕机后消息无法及时被消费）
    2. 多个 master 多个 slave
       1. master -> slave 消息同步方式为**同步**（较异步方式性能略**低**，消息**无延迟**）
       2. master -> slave 消息同步方式为**异步**（较同步方式性能略**高**，消息**略有延迟**）

~~（TODO：画图）~~

### RocketMQ 集群特征

RocketMQ 集群工作流程

步骤1：NameServer 启动，开启监听，等待 broker、producer、consumer 连接
步骤2：broker 启动，根据配置信息，连接所有的 NameServer，并保持长连接
    （补充：如果 broker 中有现存数据，NameServer 将保存 topic 与 broker 的关系）
步骤3：producer 启动，连接某个 NameServer，并建立长连接，以获取 Topic 元数据（队列列表等）
步骤4：producer 发消息
    步骤4.1：如果 topic 存在 -> 由 NameServer 直接分配
    步骤4.2：如果 topic 不存在 -> 由 NameServer 创建 topic 与 broker 的关系，并分配
步骤5：producer 从 broker 上该 Topic 对应的队列列表中选择一个目标队列用于投递消息（根据负载均衡或自定义策略）。
步骤6：producer 与 broker 建立长连接，用于发送消息
步骤7：producer 发送消息

consumer 的工作流程同 producer。

### 详细部署步骤

~~（TODO：补充）~~

## **高级特性（重点）**

### 消息存储机制

#### 消息投递与消费流程（含 ACK 机制）

① 消息生产者发送消息到 MQ
② MQ 返回 ACK 给生产者
③ MQ push 消息给对应的消费者
④ 消息消费者返回 ACK 给 MQ

~~（TODO：画图）~~

#### 消息存储机制（持久化 & 删除）

① 消息生产者发送消息到 MQ
② MQ 收到消息，将消息进行持久化，存储该消息
③ MQ 返回 ACK 给生产者
④ MQ push 消息给对应的消费者
⑤ 消息消费者返回 ACK 给 MQ
⑥ MQ 删除消息

~~（TODO：画图）~~

注意：
- 第 ⑤ 步 MQ 在指定时间内接收到消息消费者返回的ACK，则 MQ 认定消息消费成功，执行 ⑥
- 第 ⑤ 步 MQ 在指定时间内未接收到消息消费者返回的ACK，则 MQ 认定消息消费失败，重新执行 ④⑤⑥

#### 消息的存储介质

1. 数据库
   1. ActiveMQ（默认支持 JDBC 数据库作为消息存储）
   2. 缺点：数据库性能成为消息吞吐的瓶颈
2. 文件系统
   1. RocektMQ/Kafka/RabbitMQ
   2. 解决方案：采用消息刷盘机制进行数据存储
   3. 缺点：如果物理磁盘损坏，可能导致消息丢失（可通过备份、副本机制缓解）

~~（TODO：画图）~~

#### 高效的消息存储与读写方式

1. SSD
   1. 随机写（100KB/s）
   2. 顺序写（600MB/s）
2. Linux 系统发送数据的方式
（TODO：补充）
3. **“零拷贝”技术**
   1. 数据传输由传统的 **4** 次复制简化成 **3** 次复制，减少 1 次复制过程
   2. Java 语言中使用 MappedByteBuffer 类实现了该技术
   3. 要求：预留存储空间，用于保存数据（1GB 存储空间起步）

```

    ┌───────┐ ┌──────────────────┐   ┌───────┐ ┌─────────┐
    │       ↓ │                  ↓   │       ↓ │         ↓
硬盘数据--->内核态--->用户态--->网络驱动内核--->网卡--->内存数据

```

~~（TODO：画图）~~

### 消息存储结构

MQ 数据存储区域包含：
**1. 消息数据存储区域（CommitLog）**
   1. topic
   2. queueId
   3. message
**2. 消费逻辑队列**
   1. minOffset
   2. maxOffset
   3. consumerOffset
**3. 索引**
   1. key 索引
   2. 创建时间索引

### 刷盘机制

#### **同步**刷盘

1. 生产者发送消息到 MQ，MQ 接收到消息数据
2. MQ 挂起生产者发送消息的线程
3. MQ 将消息数据写入内存
4. 内存数据写入磁盘
5. 磁盘存储后返回 SUCCESS
6. MQ 恢复挂起的生产者线程
7. 发送 ACK 到生产者

~~（TODO：画图）~~

#### **异步**刷盘

1. 生产者发送消息到 MQ，MQ 接收到消息数据
2. MQ 将消息数据写入内存
3. 发送 ACK 到生产者

#### 对比

同步刷盘：安全性高，慢（适用于对数据安全要求较高的业务）
异步刷盘：安全性低，快（适用于对数据处理速度要求较高的业务）

#### 配置方式

```
# 刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
```

### 高可用性

1. nameserver
   无状态 + 全服务器注册
2. 消息服务器
   主从架构（如 2M-2S）
3. 消息生产
   生产者通过注册多个 broker，自动根据 topic 的路由信息选择可用的 master 队列发送消息（多队列容错机制），保证 broker 挂掉后仍可将消息投递至其他 broker 的可用 master 队列
4. 消息消费
   （默认由 master 提供消费拉取服务）如果启用 enableSlaveActingMaster，在 master 异常时，slave 节点可临时接管消费请求，实现 failover

### 主从数据复制机制

#### 同步复制

master 接收到消息后，先复制到 slave，然后反馈给生产者写操作成功

优点：数据安全，不丢数据，出现故障容易恢复
缺点：影响数据吞吐量，整体性能低

#### 异步复制

master 接收到消息后，立即返回给生产者写操作成功，当消息达到一定量后再异步复制到 slave

优点：数据吞吐量大，操作延迟低，性能高
缺点：数据不安全，会出现数据丢失的现象，一旦 master 出现故障，从上次数据同步到故障时间的数据将丢失

#### 配置方式

```
# Broker 的角色
#- ASYNC_MASTER     异步复制 Master
#- SYNC_MASTER      同步双写 Master
#- SLAVE            Slave 节点
brokerRole=SYNC_MASTER
```

### 负载均衡机制

#### Producer 负载均衡

我们知道，producer 在发送消息时会根据 topic 的路由信息，从多个 broker 中获取可用的消息队列（MessageQueue）列表

具体就是通过**轮询（Round-Robin）**的策略在这些队列中选择一个目标队列，实现 Producer 端的消息发送负载均衡

~~（TODO：画图）~~

#### Consumer 负载均衡

1. **平均分配策略（默认）**

将 topic 下所有消息队列平均分配给多个消费者实例（也就是说，如果队列数不能被消费者数整除，那么部分消费者会分到更多队列）

~~（TODO：画图）~~

1. **循环分配策略（如自定义实现）**

可通过扩展 AllocateMessageQueueStrategy 实现按需分配逻辑（如轮询、权重等）

默认实现是平均分配（AllocateMessageQueueAveragely）

~~（TODO：画图）~~

#### 广播模式（不参与负载均衡）

每个 Consumer 实例会消费 topic 下**全部的消息队列**。不存在队列分配逻辑，所以不涉及负载均衡。

### 消息重试机制

当消息未正常返回消费成功（ACK）时，RocketMQ 会启动消息重试机制，确保消息最终被成功消费

消息类型区分为**顺序消息**和**无序消息**

#### 顺序消息重试

顺序消息**强调顺序性**，重试逻辑特殊

当消费者消费消息失败后，RocketMQ 会每隔 **1 秒** 消息重试一次

注意：顺序消息会**阻塞所在队列的后续消息**，所以需要谨慎处理异常，同时建议对顺序消费业务进行监控，避免消息阻塞。

~~（TODO：画图）~~

#### 无序消息重试

无序消息包括：普通消息、定时消息、延时消息、事务消息（反正顺序消息之外的都是无序消息）

无序消息重试仅适用于集群消费模式（负载均衡）的消息消费，**不适用于广播模式的消息消费**

为了保障无序消息的消费，MQ 设定了**合理的消息重试间隔时长**，最多重试 **16 次**：
10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m,
9m, 10m, 20m, 30m, 1h, 2h

### 死信队列

当消息消费重试到达了指定次数（默认 16 次）后，MQ 将这些无法被正常消费的消息成为死信队列（Dead-Letter Message）

死信消息不会被直接抛弃，而是保存到一个全新的队列中，叫做死信队列（Dead-Letter Queue）

死信队列的特征：

1. 归属于某一个组（Group Id），而不归属于 Topic，也不归属于消费者
2. 一个死信队列中可以包含同一个 Group 下的多个 Topic 中的死信消息
3. 死信队列不会进行默认初始化，当第一个死信出现后，此队列被首次初始化

死信队列中消息的特征：

1. 不会被再次重复消费
2. 有效期为 **3 天**，达到时限后被清除

### 死信的处理

在监控平台中，通过查找死信，获取死信的 messageId，然后通过 id 对死信进行精准消费

### 消息重复消费

原因：

1. 生产者发送了重复的消息
   1. 网络闪断
   2. 生产者宕机
2. 消息服务器投递了重复的消息
   1. 网络闪断
3. 动态的负载均衡过程
   1. 网络闪断/抖动
   2. broker 重启
   3. 订阅方（消费者）应用重启
   4. 客户端扩容
   5. 客户端缩容

### 消息幂等

对同一条消息，无论消费多少次，结果保持一致，成为消息幂等性

解决方案：

1. 使用业务 id 作为消息的 key
2. 在消费消息时，客户端对 key 做判定，未使用过的放行，使用过的抛弃

注意：messageId 由 RocketMQ 产生，并不具备唯一性，不能用作幂等的判定条件！

常见的幂等方法示例：

- 增：**不幂等**    insert into order values (...)
- 删：**幂等**      delete from 表 where id = 1
- 改：**不幂等**    update account set **balance = balance + 100** where no = 1
- 改：**幂等**      update account set **balance = 100** where no = 1
- 查：**幂等**