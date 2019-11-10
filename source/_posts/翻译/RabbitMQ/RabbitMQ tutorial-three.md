---
title: Rabbit tutorial-three
date: 2018-12-01 23:22:11
tags: [编程,分布式,消息队列]
categories: 消息队列	
---

RabbitMQ入门教程系列---（3）

  <!-- more -->

  # RabbitMQ tutorial-three

### 发布/订阅 （Publish/Subscribe)


在上一个教程中，我们创建了一个工作队列(`Work Queues `)。 工作队列可以正常运行的背后是每个任务都被确切地交付给了一个工作者(`Worker`)。 

在这一节，我们将做一些完全不同的事情——我们将向多个消费者发送消息。 这种模式被称为"发布/订阅"。

为了演示该模式，我们将构建一个简单的日志记录系统。 它将由两个程序组成——第一个程序将发出日志消息，第二个程序将接收和打印日志消息。

在我们的日志系统中，每一个正在运行的接收程序都会得到这些消息。 这样，我们就能够收到消息，并将日志消息存到磁盘; 同时，我们还能够在另一个消费者处查看日志。

#### 交换器 (Exchanges)
在本教程的前几部分中，我们都是在队列中发送和接收消息。现在是时候引入完整的`Rabbit` 消息模型了。

让我们快速回顾一下之前的教程:
- 生产者 `Producer`: 一个发送消息的应用.
- 消息队列 `Queue`: 一个存储消息的缓冲区.
- 消费者 `Consumer`: 一个接收消息的应用.

`RabbitMQ `中消息传递模型的核心思想是:生产者从不直接向队列发送任何消息, 实际上,生产者甚至根本不知道消息是否会被传递到任何队列。

相反，生产者只能向交换器( `exchange` )发送消息。 交换是一件非常简单的事情, 它一边接收来自生产者的消息，另一边将消息推送到队列。 交换机必须确切地知道如何处理它收到的消息---- 它是否应该附加到特定的队列中？ 它是否应该附加到全部队列中？ 还是应该被抛弃。 其规则由交换器类型( `exchange type` )定义。
![exchange](http://www.rabbitmq.com/img/tutorials/exchanges.png)

`RabbitMQ`提供了几种交换器类型: 直接类型( `direct` )、主题类型( `topic` )、页眉类型( `headers ` )和扇出类型( `fanout` )。 我们将把重点放在最后一个----扇出类型。 让我们创建一个这种类型的交换，并命名为`logs`:
```java
channel.exchangeDeclare("logs", "fanout");
```
扇出交换非常简单,正如你可能从名称中猜到的那样，它只是将收到的所有消息传播到它所知道的所有队列----而这正是我们的`logger`所需要的。 
> **列出所有 Exchanges**
>
> 要列出服务器上的交换器，你可以运行曾经有用的 rabbitmqctl:
> 
> ```bash
> sudo rabbitmqctl list_exchanges
> ```
> 在这个列表中将有一些名称为 `amq.*`的`Exchanges`和默认交换器(未命名)。这些都是默认创建的，但目前不太可能需要使用它们。
> 
> **无名称的 Exchanges**
>
> 在本教程的前几部分中，我们对交换器一无所知，但仍然能够将消息发送到队列。 这是可能的，因为我们使用的是默认交换，我们通过空字符串("")来标识它。
> 
> 我们以前是怎么发布消息的:
> ```java
> channel.basicPublish("", "hello", null, message.getBytes());
> ```
> 第一个参数是交换的名称；第二个参数为空字符串，表示使用默认交换或无名称交换: 消息被路由到指定的 `routingKey` 队列中。

现在，我们可以发布到指定名称的交换器了:
```java
channel.basicPublish( "logs", "", null, message.getBytes());
```

#### 临时队列 (Temporary queues)

你可能还记得，在前面的教程里，我们使用了具有特定名称的队列(还记得` hello `和 `task queue `吗?) . 能够命名一个队列对我们来说至关重要----我们需要将工作队列指向同一个队列。 当您希望在生产者和使用者之间共享队列时，为队列赋予一个名称非常重要。

但是对于我们的`logger`来说，情况并非如此。 我们希望看到所有的日志信息，而不仅仅是其中的一个部分。 我们也只对当前信息流感兴趣，而不是旧的信息。 要解决这个问题，我们需要两样东西。

- 首先，无论何时我们连接到`RabbitMQ`的时候，我们总需要一个新的空队列。 为此，我们可以创建一个具有随机名称的队列，或者更好的做法是让服务器为我们选择一个随机队列名称。

- 其次，一旦我们断开消费者的连接，队列应该被自动删除。

在 `Java `中，当我们不向 `queueDeclare ()`提供任何参数时，我们就创建了一个非持久的、独占的、自动删除的队列，队列名称可以这样获得:
```java
String queueName = channel.queueDeclare().getQueue();
```
此时 `queueName` 包含随机队列名,它可能看起来像 `amq.gen-jzty20brgko-hjmuj0wlg。`

#### 绑定 （Bindings）
![bingding](http://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了扇出交换( `fanout` )和队列 (`queue`) 。 现在我们需要告诉交换器将消息发送到我们的队列,交换器和队列之间的关系称为**绑定**。
```java
channel.queueBind(queueName, "logs", "");
```
从现在起，日志交换器将向我们的队列追加消息。
> 列出所有绑定( `bindings ` )
> ```bash
> rabbitmqctl list_bindings
> ```

#### 将以上步骤聚合
![together](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

生产者程序发出(`emit`)日志信息，看起来和上一个教程没有什么不同。 最重要的变化是，我们现在想要将消息发布到`log`交换器中，而不是发布无名消息。 我们需要在发送时提供一个 `routingKey`，但是对于扇出交换器它的值被忽略。 下面是 `EmitLog.java `程序的代码:
```java
import java.io.IOException;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class EmitLog {

    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] argv)
                  throws java.io.IOException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
    //...
}
```

如你所见，在建立连接后，我们宣布交换。 这一步是必要的，因为发布到不存在的交换是被禁止的。

如果还没有队列绑定到交换器，消息将丢失，但这对我们来说是可以的; 如果还没有消费者在监听，我们可以安全地丢弃消息。

以下是 `ReceiveLogs.java ` :
```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogs {
  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName, EXCHANGE_NAME, "");

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope,
                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
      }
    };
    channel.basicConsume(queueName, true, consumer);
  }
}
```

> 本教程全文翻译自 `http://www.rabbitmq.com/tutorials/tutorial-three-java.html`