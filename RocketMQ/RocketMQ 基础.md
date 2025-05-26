# RocketMQ 基础知识

https://rocketmq.apache.org/

## 为什么使用 MQ？-> MQ 的优势和劣势

优势（作用）：
1. 应用解耦
2. 异步提速
3. 削峰填谷

劣势：
1. 系统可用性降低
2. 系统复杂度提高
3. 异步消息机制
   1. 消息顺序性
   2. 消息丢失
   3. 消息一致性
   4. 消息重复使用

应用解耦：消费方存活与否不影响生产方
系统的耦合性越高，容错性就越低，可维护性就越低。

## 主流 MQ 产品对比

1. ActiveMQ：Java 语言实现，万级数据吞吐量，处理速度 ms 级，主从架构，**成熟度高**
2. RabbitMQ：Erlang 语言实现，万级数据吞吐量，**处理速度 us 级**，主从架构
3. RocketMQ：Java 语言实现，**十万级**数据吞吐量，分布式架构，功能强大，扩展性强
4. Kafka：Scala 语言实现，**十万级**数据吞吐量，处理速度 ms 级，分布式架构，功能较少，应用于大数据较多

## RocketMQ 架构图

（TODO：画图）

## 环境搭建

略。

## **消息发送（重点）**

### 主要内容

1. 基于 Java 环境构建发送与消息接受基础程序
   1. 单生产者-单消费者
   2. 单生产者-多消费者
   3. 多生产者-多消费者
2. 发送不同类型的消息
   1. 同步消息
   2. 异步消息
   3. 单向消息
3. 特殊的消息发送
   1. 延时消息
   2. 批量消息
4. 特殊的消息接收
   1. 消息过滤
5. 消息发送与接收顺序控制
6. 事务消息

### 消息发送、消息接收的开发流程

   1. 谁发
   2. 发给谁
   3. 怎么发
   4. 发什么
   5. 发的结果是什么
   6. 打扫战场

### 关系

1. One-To-One（基础发送与基础接收）
2. One-To-Many（负载均衡模式与广播模式）
3. Many-To-Many

### 单生产者-单消费者（One-To-One）

1. 生产者

```Java
public class Producer {
   public static void main(String[] args) throws Exception {
      /**
        * 1. 谁发
        * 2. 发给谁
        * 3. 怎么发
        * 4. 发什么
        * 5. 发的结果是什么
        * 6. 打扫战场
        */

      // 1. 创建一个发送消息的对象 Producer
      DefaultMQProducer producer = new DefaultMQProducer("group1");
      // 2. 设定发送的命名服务器地址
      producer.setNamesrvAddr("localhost:9876");
      // 3.1 启动发送的服务
      producer.start();
      // 4. 创建要发送的消息对象，指定 topic，指定内容 body
      Message msg = new Message("topic1", "hello rocketmq".getBytes("UTF-8"));
      // 3.2 发送消息
      SendResult result = producer.send(msg);
      System.out.println("返回结果：" + result);
      // 5. 关闭连接
      producer.shutdown();
   }
}
```

2. 消费者

```Java
public class Consumer {
   public static void main(String[] args) throws Exception {
      /**
        * 1. 谁发
        * 2. 发给谁
        * 3. 怎么发
        * 4. 发什么
        * 5. 发的结果是什么
        * 6. 打扫战场
        */

      // 1. 创建一个接收消息的对象 Consumer
      DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
      // 2. 设定接收的命名服务器地址
      consumer.setNamesrvAddr("localhost:9876");
      // 3.1 设置接收消息对应的topic，对应的sub标签为任意
      consumer.subscribe("topic1", "*");
      // 3.2 开启监听，用于接收消息
      consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
               // 遍历消息
               for (MessageExt msg : list) {
                  System.out.println("收到消息："+msg);
               }
               return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
      });
      // 4. 启动接收消息的服务
      consumer.start();
      System.out.println("接受消息服务已开启");
      // 5. 注意：不要关闭消费者！
   }
}
```

### 单生产者-多消费者（One-To-Many）：**负载均衡**

1. 生产者

```Java
// 1. 创建一个发送消息的对象 Producer
DefaultMQProducer producer = new DefaultMQProducer("group1");
// 2. 设定发送的命名服务器地址
producer.setNamesrvAddr("localhost:9876");
// 3.1 启动发送的服务
producer.start();
for (int i = 0; i < 10; i++) {
   // 4. 创建要发送的消息对象，指定 topic，指定内容 body
   Message msg = new Message("topic1", ("hello rocketmq" + i).getBytes("UTF-8"));
   // 3.2 发送消息
   SendResult result = producer.send(msg);
   System.out.println("返回结果：" + result);
}
//5.关闭连接
producer.shutdown();
```

2. 消费者**（默认模式：负载均衡）**

开启多实例运行：
Edit Configurations - Add Run Options - Allow multiple instances

说明：同一个消费者，多个实例，争抢 topic 数据。

```Java
// 1. 创建一个接收消息的对象 Consumer
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
// 2. 设定接收的命名服务器地址
consumer.setNamesrvAddr("localhost:9876");
// 3.1 设置接收消息对应的 topic，对应的 sub 标签为任意
consumer.subscribe("topic1", "*");
// 设置当前消费者的消费模式（默认模式：负载均衡）
consumer.setMessageModel(MessageModel.CLUSTERING);
// 3.2 开启监听，用于接收消息
consumer.registerMessageListener(new MessageListenerConcurrently() {
   @Override
   public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
         // 遍历消息
         for (MessageExt msg : list) {
            System.out.println("收到消息：" + msg);
            System.out.println("消息是：" + new String(msg.getBody()));
         }
         return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
   }
});
// 4. 启动接收消息的服务
consumer.start();
System.out.println("接受消息服务已开启");

// 5. 注意：不要关闭消费者！
```

### 单生产者-多消费者（One-To-Many）：**广播模式**

1. 生产者

同上。

2. 消费者**（广播模式）**

```Java
// 1. 创建一个接收消息的对象 Consumer
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
// 2. 设定接收的命名服务器地址
consumer.setNamesrvAddr("localhost:9876");
// 3.1 设置接收消息对应的 topic，对应的 sub 标签为任意
consumer.subscribe("topic1","*");
// 设置当前消费者的消费模式（默认模式：负载均衡）
// consumer.setMessageModel(MessageModel.CLUSTERING);
// 设置当前消费者的消费模式（广播模式）
consumer.setMessageModel(MessageModel.BROADCASTING);
// 3.2 开启监听，用于接收消息
consumer.registerMessageListener(new MessageListenerConcurrently() {
   @Override
   public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
         //遍历消息
         for (MessageExt msg : list) {
            System.out.println("收到消息：" + msg);
            System.out.println("消息是：" + new String(msg.getBody()));
         }
         return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
   }
});
// 4. 启动接收消息的服务
consumer.start();
System.out.println("接受消息服务已开启");

// 5. 注意：不要关闭消费者！
```

### 多生产者-多消费者（Many-To-Many）

多生产者产生的消息可以被同一个消费者消费，也可以被多个消费者消费。

## 消息类别

1. 同步消息
2. 异步消息
3. 单向消息

### 同步消息

特征：即时性较强，重要，且必须有回执的消息，如*短信、通知（转账成功）*

```Java
SendResult result = producer.send(msg);
```

### 异步消息

特征：即时性较弱，但需要有回执的消息，如*订单中的某些信息*

```Java
// 1. 创建一个发送消息的对象 Producer
DefaultMQProducer producer = new DefaultMQProducer("group1");
// 2. 设定发送的命名服务器地址
producer.setNamesrvAddr("localhost:9876");
// 3.1 启动发送的服务
producer.start();
for (int i = 0; i < 10; i++) {
   // 4. 创建要发送的消息对象，指定 topic，指定内容 body
   Message msg = new Message("topic1", ("hello rocketmq"+i).getBytes("UTF-8"));
   // 3.2 同步消息
   // SendResult result = producer.send(msg);
   // System.out.println("返回结果：" + result);

   // 异步消息
   producer.send(msg, new SendCallback() {
         // 表示成功返回结果
         @Override
         public void onSuccess(SendResult sendResult) {
            System.out.println(sendResult);
         }
         // 表示发送消息失败
         @Override
         public void onException(Throwable throwable) {
            System.out.println(throwable);
         }
   });
   
   System.out.println("消息" + i + "发完了，我先做业务逻辑去了~");
}
// 休眠 10s
TimeUnit.SECONDS.sleep(10);
// 5. 关闭连接
producer.shutdown();
```

### 单向消息

特征：不需要有回执的消息，如*日志类消息*

```Java
producer.sendOneway(msg);
```

### 延时消息

消息发送时并不直接发送到消息服务器，而是根据设定的等待时间到达，起到延时到达的缓冲作用。

```Java
Message msg = new Message("topic3", ("延时消息：hello rocketmq " + i).getBytes("UTF-8"));
// 设置延时等级 3，这个消息将在 10s 之后发送（现在只支持固定的几个时间，详看 delayTimeLevel）
msg.setDelayTimeLevel(3);
SendResult result = producer.send(msg);
System.out.println("返回结果：" + result);
```

目前支持的消息时间

```Java
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
```

### 批量消息

批量发送消息能显著提高传递小消息的性能。

```Java
List<Message> msgList = new ArrayList<Message>();
Message msg1 = new Message("topic1", ("hello rocketmq1").getBytes("UTF-8"));
Message msg2 = new Message("topic1", ("hello rocketmq2").getBytes("UTF-8"));
Message msg3 = new Message("topic1", ("hello rocketmq3").getBytes("UTF-8"));

msgList.add(msg1);
msgList.add(msg2);
msgList.add(msg3);

SendResult result = producer.send(msgList);
```

注意限制：

1. 这些批量消息应该有相同的 topic
2. 这些批量消息应该有相同的 waitStoreMsgOK
3. 不能是延时消息
4. 消息内容总长度不超过 4 MB

消息内容总长度包含如下：

1. topic（字符串字节数）
2. body（字节数组长度）
3. 消息追加的属性（key 与 value 对应字符串字节数）
4. 日志（固定 20 字节）

### 消息过滤

#### 分类过滤

按照 tag 过滤消息。

生产者

```Java
Message msg = new Message("topic6", "tag2", ("消息过滤按照tag：hello rocketmq 2").getBytes("UTF-8"));
```

消费者

```Java
// 接收消息的时候，除了制定 topic，还可以指定接收的 tag，* 代表任意 tag
consumer.subscribe("topic6", "tag1 || tag2");
```

#### 语法过滤（属性过滤/语法过滤/SQL过滤）

基本语法与 SQL 类似：

- 数值比较，如：>, >=, <, <=, BETWEEN, =
- 字符比较，如：=, <>, IN
- IS NULL 或者 IS NOT NULL
- 逻辑符号：AND, OR, NOT

常量支持类型：

- 数值，如：123, 3.1415
- 字符，如：'abc'（必须用单引号包裹起来）
- NULL（特殊的常量）
- 布尔值：TRUE 或 FALSE

生产者

```Java
// 为消息添加属性
msg.putUserProperty("vip", "1");
msg.putUserProperty("age", "20");
```

消费者

```Java
// 使用消息选择器来过滤对应的属性，语法格式为类 SQL 语法
consumer.subscribe("topic7", MessageSelector.bySql("age >= 18"));
consumer.subscribe("topic6", MessageSelector.bySql("name = 'litiedan'"));
```

注意：SQL 过滤需要依赖服务器的功能支持，在 broker.conf 配置文件中添加对应的功能项，并开启对应功能

```properties
enablePropertyFilter=true
```

重启 broker

start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true

或者直接在终端输入

```
mqadmin.cmd updateBrokerConfig -blocalhost:10911 -kenablePropertyFilter -vtrue
```

然后可以在 RocketMQ 控制台（需安装）查看开启与否

## Spring Boot 整合

### 导包

略。

### 配置文件

```properties
rocketmq.name-server=localhost:9876
rocketmq.producer.group=demo_producer
```

### 实体类

```Java
public class user implements Serializable {
   String userName;
   String userId;

   public user(){

   }

   public user(String userName, String userId) {
      this.userName = userName;
      this.userId = userId;
   }

   @Override
   public String toString() {
      return "demoEntity{" +
               "userName='" + userName + '\'' +
               ", userId='" + userId + '\'' +
               '}';
   }
}
```

### 生产者

```Java
@RestController
public class DemoProducers {
   @Autowired
   private RocketMQTemplate template;

   @RequestMapping("/producer")
   public String producersMessage() {
      User user = new User("fan", "123456789");
      template.convertAndSend("demo-topic", user);
      return JSON.toJSONString(user);
   }
}
```

### 消费者

```Java
@Service
@RocketMQMessageListener(topic = "demo-topic", consumerGroup = "demo_consumer")
public class DemoConsumers1 implements RocketMQListener<user> {
   @Override
   public void onMessage(user user) {
      System.out.println("Consumers1接收消息:" + demoEntity.toString());
   }
}
```

### 其他消息

#### 异步发送

```Java
rocketMQTemplate.asyncSend("topic9", user, new SendCallback() {
   @Override
   public void onSuccess(SendResult sendResult) {
      System.out.println(sendResult);
   }

   @Override
   public void onException(Throwable throwable) {
      System.out.println(throwable);
   }
});
```

#### 单向发送

```Java
rocketMQTemplate.sendOneWay("topic9",user);
```

#### 延时消息

```Java
rocketMQTemplate.syncSend("topic9", MessageBuilder.withPayload("test delay").build(), 2000, 2);
```

#### 批量

```Java
List<Message> msgList = new ArrayList<>();
msgList.add(new Message("topic6", "tag1", "msg1".getBytes()));
msgList.add(new Message("topic6", "tag1", "msg2".getBytes()));
msgList.add(new Message("topic6", "tag1", "msg3".getBytes()));
rocketMQTemplate.syncSend("topic8", msgList, 1000);
```

#### tag 过滤

消费者

```Java
@RocketMQMessageListener(topic = "topic9", consumerGroup = "group1", selectorExpression = "tag1")
```

SQL 过滤

```Java
@RocketMQMessageListener(topic = "topic9", consumerGroup = "group1", selectorExpression = "age>18", selectorType= SelectorType.SQL92)
```

改消息模式

```Java
@RocketMQMessageListener(topic = "topic9", consumerGroup = "group1", messageModel = MessageModel.BROADCASTING)
```