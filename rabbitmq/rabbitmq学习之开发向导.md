# 一、连接RabbitMQ

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

