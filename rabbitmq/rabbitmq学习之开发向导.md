一、连接RabbitMQ

连接RabbitMQ有两种方式，如下

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(USERNAME);
factory.setPassword(PASSWORD);
factory.setVirtualHost(VIRTUALHOST);
factory.setHost(HOST);
factory.setPort(PORT);
Connection connection = factory.newConnection();
```

也可以按照如下方式连接

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://username:password@host:port/virtualHost");
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
```

# 二、使用交换器和队列

```java
channel.exchangeDeclare(exchangeName,"direct",true);
channel.queueDeclare(queueName,true,false,false,null);
channel.queueBind(queueName,exchangeName,routingkey);
```

上面的代码生命了一个持久化的、非自动删除订单、绑定类型为direct类型的交换器，同时创建了一个持久化的、排他的、非自动删除的、被分配了一个确定名称的队列

# 三、消费消息

RabbitMQ的消费模式分两种：推模式和拉模式

**注意：**RabbitMQ的推模式试讲信道设置为投递模式，知道取消队列的订阅为止。在投递模式期间，RabbitMQ会不断地推送消息给消费者，当然推送消息的个数还是会受到Basic.Qos的限制。如果只想从队列获得单条消息而不是持续订阅，建议还是使用拉模式进行消费。但是不能将拉模式放在一个循环里代替推模式，这样做会严重影响RabbitMQ的性能。如果要实现高吞吐量，消费者理应使用推模式。

# 四、消费端的确认与拒绝

为了确保消息从队列可靠地达到消费者，RabbitMQ提供了消息确认机制。消费者在订阅队列时，可以指定autoAck参数，当autoAck等于false时，RabbitMQ会等待消费者显式地恢复确认信号后才从内存（或磁盘）中移去消息（实质上是先打上删除标记，之后再删除）。当autoAck等于true时，RabbitMQ会自动把发送出去的消息置为确认，然后从内存（或磁盘）中删除，而不管消费者是否真正地消费到了这些消息

