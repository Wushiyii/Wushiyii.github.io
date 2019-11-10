---
  title: Rabbit tutorial-two
  date: 2018-11-30 10:24:18
  tags: [编程,分布式,消息队列]
  categories: 消息队列	
---

RabbitMQ入门教程系列---（2）

  <!-- more -->

  # RabbitMQ tutorial-two

### 工作队列(Work Queues)

在第一个教程中，我们编写了从命名队列、发送和接收消息的程序。 在本例中，我们将创建一个工作队列(`Work Queue`)，用于在多个工作者(`worker`)之间分配耗时的任务。

工作队列`Work Queue`(又名: 任务队列`Task Queue`)的主要思想是避免立即执行资源密集型任务并等待完成。 我们将任务封装为消息，并将其发送到队列。 在后台运行的工作进程（`worker process`）将弹出任务并最终执行作业。 当您运行许多工作者(`Worker`)时，任务将在他们之间共享。



#### 准备工作

在本教程的第一部分中，我们发送了一条包含"Hello World!"的消息。 现在我们将使用 `Thread.sleep ()`模拟任务执行中的时间花费， 我们将字符串中的点的数量作为它的时间复杂性---- 每个点将占用"工作"的一秒钟。 例如，Hello... 描述的虚假任务将花费三秒钟。

我们将对上一个示例中的` Send.java` 代码稍作修改，以允许从命令行发送任意消息。 这个程序将把任务安排到我们的工作队列中，所以让我们把它命名为 `NewTask.java`:

```java
String message = getMessage(new String[]{"hello ..."});

channel.basicPublish("", "hello", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```

处理字符串的函数

```java
private static String getMessage(String[] strings){
    if (strings.length < 1)
        return "Hello World!";
    return joinStrings(strings, " ");
}

private static String joinStrings(String[] strings, String delimiter) {
    int length = strings.length;
    if (length == 0) return "";
    StringBuilder words = new StringBuilder(strings[0]);
    for (int i = 1; i < length; i++) {
        words.append(delimiter).append(strings[i]);
    }
    return words.toString();
}
```

现在新建一个基于`Recv`的实例`Worker`，增加一些处理字节的函数: 它将接收每个byte，通过函数判断进行延迟时间以模拟执行任务。

```java
final Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
    String message = new String(body, "UTF-8");

    System.out.println(" [x] Received '" + message + "'");
    try {
      doWork(message);
    } finally {
      System.out.println(" [x] Done");
    }
  }
};
boolean autoAck = true; // acknowledgment is covered below
channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
```

修改`DefaultConsumer`中实现的处理方式

```java
private static void doWork(String task) throws InterruptedException {
    for (char ch: task.toCharArray()) {
        if (ch == '.') Thread.sleep(1000);
    }
}
```

####  使用循环调度

使用任务队列的优点之一是能够轻松地并行处理工作。 如果我们正在建立积压的工作，我们可以只增加更多的工人（`Worker`）来扩展系统。

首先，让我们尝试同时运行两个工作实例。 它们都将从队列获取消息。

接下来，启动`NewTask`发布消息，

```bash
[x] Sent 'First message .'
[x] Sent 'Second message ..'
[x] Sent 'Third message ...'
[x] Sent 'Four message ....'
[x] Sent 'Fifth message .....'
```

`Worker`处接收到的响应:

```bash
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'

java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，`RabbitMQ `将按顺序将每条消息发送给下一个消费者。 平均而言，每个消费者都将获得相同数量的消息。 这种分发消息的方式称为`round-robin`循环。



#### 消息消费确认

完成一项任务可能需要几秒钟。 你可能想知道，如果一个消费者开始了一个长时间的任务，并且只完成了一部分就挂掉了，会发生什么？ 使用我们当前的代码，一旦` RabbitMQ` 向用户发送消息，它将立即将其标记为删除。 在这种情况下，如果挂掉一个` worker`，我们将丢失它正在处理的消息。 我们还将丢失所有发送给这个特定的工作者但尚未处理的消息。

但我们不想丢失掉任何消息， 如果一个`worker`挂了，我们希望把任务传递给另一个`worker`。

为了确保消息不会丢失，`RabbitMQ` 支持消息确认。 一个 `ack`状态码被消费者发送回来，告诉 `RabbitMQ` 一条特定消息已经被接收和处理，`RabbitMQ `可以自由地删除它。

如果使用者没有发送一个 `ack `就挂了(由于通道被关闭，连接被关闭，或者 TCP 连接丢失等原因) ，`RabbitMQ` 将会将对它进行重新队列。 如果有其他消费者在同一时间在线，它将很快重新交付给另一个消费者。 这样你可以确保消息不会丢失，即使`worker`偶尔挂掉。

在默认情况下，`RabbitMQ`需要手动设置消息确认。 在前面的示例中，我们通过 `autoAck` 的 `true `标志明确关闭了它们。现在我们将这个标志设置为 `false`，并发送一个来自工作者`worker`的适当确认信息。

```java
channel.basicQos(1); // 一次只接收一次无`ack`标志的消息

final Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
    String message = new String(body, "UTF-8");

    System.out.println(" [x] Received '" + message + "'");
    try {
      doWork(message);
    } finally {
      System.out.println(" [x] Done");
      channel.basicAck(envelope.getDeliveryTag(), false);
    }
  }
};
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
```

>  忘记添加ACK标志
>
> 没有添加`Ack`标志。 这是一个简单的错误，但后果是严重的。 当你的客户端退出的时候，信息是会被重新传递的 并被重复消费，但并且RabbitMQ 会消耗更多的内存，因为它不能释放任何未删除的信息。
>
> 为了调试此类错误，可以使用 `rabbitmqctl `打印未确认字段的消息:
>
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
>
> 在 Windows 中，删除 sudo:
>
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged

#### 消息持久化

我们已经学会了如何确保即使消费者死亡，这项任务也不会丢失。 但是，如果 `RabbitMQ `服务器停止运行，我们的任务仍然会丢失。

当` RabbitMQ `退出或崩溃时，它将清空队列和消息，除非设置MQ不要这样做。 要确保消息不丢失，需要做两件事: 将队列和消息都标记为持久。

首先，我们需要确保 `RabbitMQ` 永远不会丢失我们的队列。 为了做到这一点，我们需要声明它是持久的:

```bash
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

尽管这个命令是正确的，但是它在我们目前的设置中不起作用。 这是因为我们已经定义了一个名为 hello 的队列，它不是持久的。 `Rabbitmq`不允许使用不同的参数重新定义现有队列。 有一个快速的解决方案——使用不同的名称声明一个队列，例如声明一个新的任务队列:

```bash
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```

`queueDeclare`的更改需要都应用到生产者代码和消费者代码。

此时，我们确信即使 `RabbitMQ `重新启动，任务队列也不会丢失。 现在我们需要通过将 `MessageProperties` (implements BasicProperties)设置为 `PERSISTENT_TEXT_PLAIN` 来将消息标记为持久化。

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

> **关于消息持久化**
>
> 将消息标记为持久化并不能完全保证消息不会丢失，尽管现在 `RabbitMQ`会将消息保存到磁盘，但在 `RabbitMQ `接受消息且尚未保存消息时，仍有一个短暂的时间窗口。 此外，`RabbitMQ `不会对每条消息都执行 `fsync` (2)——可能只是将其保存到缓存中，而不会真正写入磁盘。 持久性保证并不强，但对于我们的简单任务队列来说已经足够了。 如果确实需要更号的持久化保证，那么你可以使用发行版。

#### 公平调度

你可能已经注意到，调度仍然没有完全按照我们所希望的那样工作。 例如，在一个有两个工作人员的情况下，当所有奇怪的信息都很重而偶数的信息都很轻时，一个工作人员会一直很忙，而另一个几乎不做任何工作。 好吧，`RabbitMQ `对此一无所知，但它仍然会平均地发送消息。

这是因为` RabbitMQ `只是在消息进入队列时分发消息。 它没有查看消费者未确认的消息数量。 它只是盲目地向第 n 个消费者发送第 n 个消息。

![fair](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

为了解决这个问题，我们采用设置了`prefetchCount = 1` 的`basicQos `方法。 这将告诉 RabbitMQ 在同一时间内不要给一个 worker 发送超过一条消息。 换句话说，在工作人员处理并确认前一条消息之前，不会向其发送新消息。 

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

#### 完整例子

` NewTask.java`

```java
import java.io.IOException;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.MessageProperties;

public class NewTask {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv)
                      throws java.io.IOException {

    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

    String message = getMessage(argv);

    channel.basicPublish( "", TASK_QUEUE_NAME,
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
    System.out.println(" [x] Sent '" + message + "'");

    channel.close();
    connection.close();
  }
  //...
}
```

`worker`

```java
import com.rabbitmq.client.*;

import java.io.IOException;

public class Worker {
  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    final Connection connection = factory.newConnection();
    final Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    channel.basicQos(1);

    final Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
          doWork(message);
        } finally {
          System.out.println(" [x] Done");
          channel.basicAck(envelope.getDeliveryTag(), false);
        }
      }
    };
    boolean autoAck = false;
    channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
  }

  private static void doWork(String task) {
    for (char ch : task.toCharArray()) {
      if (ch == '.') {
        try {
          Thread.sleep(1000);
        } catch (InterruptedException _ignored) {
          Thread.currentThread().interrupt();
        }
      }
    }
  }
}
```
> 本教程全文翻译自 `http://www.rabbitmq.com/tutorials/tutorial-two-java.html`
