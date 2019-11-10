---
  title: Rabbit tutorial-Five
  date: 2018-12-4 09:03:23
  tags: [编程,分布式,消息队列]
  categories: 消息队列	
---

RabbitMQ入门教程系列---（5）

  <!-- more -->

  # RabbitMQ tutorial-Five

### 路由(Routing)

在上一个教程中，我们使用直接交换器改进了我们的日志记录系统，从而获得了有选择地接收日志的可能性。

虽然使用直接交换改善了我们的系统，它仍然有局限性———它不能根据多个标准进行路由。

我们的日志系统中，我们可能希望不仅基于日志级别，而且也可以基于发出日志的源类型来订阅日志。类似的概念可以在 `syslog`的 `unix `工具中了解，该工具根据日志级别 (info / warn / crit...)和不同设备类型(auth / cron / kern...)对日志进行路由。

这将给我们带来很大的灵活性——我们可以只监听来自"cron"的关键错误信息，同时也可以监听所有来自"kern"的日志。

要在日志系统中实现这一功能，我们需要了解更复杂的主题( `Topic` )交换。

#### 主题交换(Topic Exchange)

任何发送到主题交换器的消息，不能有任何`routing_key`----它必须是一个用点分隔的名称列表。 这些词可以是任何东西，但通常它们指定了一些与消息相关的特征。这是一些有效的路由关键示例:`stock.usd.nyse`、`nyse.vmw`、`quick.orange.rabbit`。 在路由键中可以有任意多的字，最多255个字节。

绑定键也必须采用同样的形式。主题交换背后的逻辑类似于直接交换(Direct exchange)——使用特定路由键发送的消息将被传递到与匹配绑定键绑定的所有队列。 然而，对于`binding keys`有两个重要的特殊情况:

- `*` (星星)可以代替正好一个字
- `#` (散列)可以替代零个或更多的单词

用一个例子来解释是最简单的:

![topic](http://www.rabbitmq.com/img/tutorials/python-five.png)

在这个例子中，我们要发送的消息都是描述动物的。 消息将以由三个单词(两个点)组成的路由键`routing key`来发送。 路由键中的第一个单词描速度，第二个单词是颜色，第三个单词是物种.

我们创建了三个绑定: `Q1`绑定`*.orange.*` ,Q2 绑定 `*.*.rabbit` 和 `lazy.#`.

这些绑定可以概括为:

- `Q1`对所有的橙色动物都感兴趣
- `Q1`想知道关于兔子的一切，还有关于懒惰动物的一切

将路由密钥设置为`quick.orange.rabbit`的消息将被传递到两个队列。 消息`lazy.orange.elephant`也将传递给他们。 另一方面,`quick.orange.fox`只能传递给第一队，而`lazy.brown.fox`只能传递给第二队。 `lazy.pink.rabbit`只会送到第二个队列一次即使它匹配两个绑定。 "quick.brown.fox"不符合任何绑定，所以它将被丢弃。

如果我们违反规则，发送了一个或四个字的短信，比如`orange`或`quick.orange.male.rabbit`，会发生什么？ 这些消息与任何绑定都不匹配，并将丢失。

另一方面,`lazy.orange.male.rabbit`，即使有四个单词，也会匹配最后一个绑定，并被送到第二个队列。

> 主题交换 (Topic Exchange)
> 
> 主题交换功能强大，可以像其他交换一样运作。
> 
> 当队列与"#"(散列)绑定键绑定时，它将接收所有消息，就像扇出交换( `fanout exchange` )一样。
>
> 当绑定中不使用特殊字符"*"(星号)和"#"(散列)时，主题交换与直接交换( `direct exchange` )一样。

#### 整合一起

我们将在日志系统中使用主题交换，首先假设日志的路由密钥有两个关键字: `<facility>.<severity>`。

本节代码与前面的教程中的代码几乎相同,这是`EmitLogTopic.java`:
```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class EmitLogTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv)
                  throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "topic");

        String routingKey = getRouting(argv);
        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
        System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");

        connection.close();
    }
    //...
}
```

这是`ReceiveLogsTopic.java`:
```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogsTopic {
  private static final String EXCHANGE_NAME = "topic_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "topic");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1) {
      System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
      System.exit(1);
    }

    for (String bindingKey : argv) {
      channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
    }

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope,
                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
      }
    };
    channel.basicConsume(queueName, true, consumer);
  }
}
```

> 本教程全文翻译自 `http://www.rabbitmq.com/tutorials/tutorial-five-java.html`