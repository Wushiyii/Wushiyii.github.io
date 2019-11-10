---
  title: Rabbit tutorial-Four
  date: 2018-12-3 11:29:48
  tags: [编程,分布式,消息队列]
  categories: 消息队列	
---

RabbitMQ入门教程系列---（4）

  <!-- more -->

  # RabbitMQ tutorial-Four

### 路由(Routing)

在上一个教程中，我们构建了一个简单的日志记录系统。 我们可以向许多接收者广播日志消息。

在本教程中，我们将向其添加一个特性----我们将使订阅仅限于消息的一个子集(部分)。 例如，我们将能够只将关键的错误消息导向日志文件(以节省磁盘空间) ，同时仍然能够在控制台上打印所有的日志消息。


#### 绑定(Bindings)

在前面的示例中，我们已经创建了绑定:

```java
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

绑定就是绑定交换器和队列之间的关系。这可以简单地理解为: 队列对此交换器中的消息感兴趣。

绑定可以采用额外的 `routingKey` 参数。 为了避免与`basic_publis`参数的混淆，我们将其称为 `binding key`。 这就是我们如何创建一个关键字绑定:
```java
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```
绑定键的含义取决于交换类型,我们之前使用的扇出交换`fanout`是默认忽略绑定的。

#### 直接交换(Direct exchange)

前一教程中的日志记录系统,将所有消息广播给使用者,我们希望将此扩展到允许根据消息的严重程度对消息进行过滤。例如，我们可能希望将日志消息系统在写入磁盘的时候，只接收关键错误`error`，而不是在警告`warning`或信息`info`日志消息上浪费磁盘空间。

之前我们使用的是扇出交换`fanout exchange`，它不能给我们很大的灵活性，它只能进行盲目的广播。

我们将使用直接交换`direct exchange`代替。直接交换背后的路由算法非常简单----一条消息到达队列时，首先判断队列的`binding key`是否与消息的`routing key相同`。

![routing](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)


在上图中，我们可以看到两个队列绑定到它的直接交换器`x`。 第一个队列绑定`orange`，第二个队列有两个绑定，一个绑定`black`，另一个绑定`green`。

在这种绑定设置中，`routing key`为`orange`的消息将被路由到队列 `Q1`， 为`black`或`green`的消息将发送到`Q2`，所有其他消息都将被丢弃。

#### 多个绑定 (Multiple bindings)
![mutiple](http://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

使用相同的绑定键绑定多个队列是完全没问题的。在我们的示例中，我们可以在 `x` 和` Q1`之间添加绑定键 black。 在这种情况下，直接交换将像扇出一样工作，并将消息传播到所有匹配的队列。 所有带有`routing key`为`black`的消息将被传递到`Q1`和`Q2`。

#### 发送日志 (Emitting logs)

我们将把这个模型用于我们的日志系统，此外，我们将消息发送到一个直接的交换器中，而不是扇出交换器。 我们将提供日志级别作为路由密钥。 这样，日志接收程序就能够选择它想要接收的日志。

首先创建一个交换:

```java
channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
```

为了简化，我们假设日志级别程度可以是`info`、`warning`、`error`之一。

#### 订阅 (Subscribing)

接收消息的工作原理与前面的教程相同，只有一个例外——我们将为我们感兴趣的每个日志级别创建一个新的绑定。

```java
String queueName = channel.queueDeclare().getQueue();

for(String severity : argv){
  channel.queueBind(queueName, EXCHANGE_NAME, severity);
}
```

#### 合并到一起

![together](http://www.rabbitmq.com/img/tutorials/python-four.png)

`EmitLogDirect.java`的代码：
```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class EmitLogDirect {

    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] argv)
                  throws java.io.IOException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "direct");

        String severity = getSeverity(argv);
        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
        System.out.println(" [x] Sent '" + severity + "':'" + message + "'");

        channel.close();
        connection.close();
    }
    //..
}
```
` ReceiveLogsDirect.java`:
```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogsDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "direct");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1){
      System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
      System.exit(1);
    }

    for(String severity : argv){
      channel.queueBind(queueName, EXCHANGE_NAME, severity);
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
> 本教程全文翻译自 `http://www.rabbitmq.com/tutorials/tutorial-four-java.html`