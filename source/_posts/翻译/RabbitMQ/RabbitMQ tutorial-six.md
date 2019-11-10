---
  title: Rabbit tutorial-Six
  date: 2018-12-4 11:29:48
  tags: [编程,分布式,消息队列]
  categories: 消息队列	
---

RabbitMQ入门教程系列---（6）

  <!-- more -->

  # RabbitMQ tutorial-Six

在第二个教程中，我们学习了如何使用工作队列( `Work Queue` )在多个工作者之间分配耗时的任务。

但是如果我们需要在远程计算机上运行一个函数并等待结果呢？ 那就另当别论了。这种模式通常称为远程过程调用(`Remote Procedure Call,RPC`)。

在本节教程中，我们将使用`RabbitMQ`来构建`RPC`系统: 一个客户端和一个可扩展的`RPC`服务器。 因为我们没有任何耗时的任务，所以我们将创建一个虚拟的、返回斐波纳契数字的`RPC`服务。

#### 客户端接口 (Client interface)

为了说明如何使用`RPC`服务，我们将创建一个简单的客户端， 它将公开一个名为`call`的方法，该方法将发送一个`RPC`请求并阻塞( `block` )，直到收到回复:
```java
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();
String result = fibonacciRpc.call("4");
System.out.println( "fib(4) is " + result);
```

> 关于 `RPC` 的一个说明
> 
> 虽然`RPC`在系统中是一种很常见的模式，但它经常受到批评。当程序员不知道函数调用是一个本地的，还是缓慢的远程`RPC`调用，问题就出现了。 这样的混乱会导致不可预知的系统，增加调试的不必要的复杂性。 错误使用 RPC 将导致不可维护的意大利面条式代码，而不是简化软件。
> 
> 考虑到这一点，请考虑以下建议:
>
> - 确保哪个函数调用是本地的，哪个是远程的。
> - 记录你的系统, 明确组件之间的依赖关系。
> - 处理错误情况。 当 RPC 服务器长时间关闭时，客户机应该如何反应？
>
> 当有疑问时，避免使用 `RPC` 。 如果可以的话，您应该使用异步管道( `asynchronous pipeline` )——而不是 `RPC` 式的阻塞。异步管道可以将结果推送到下一个计算的阶段，而不用一直阻塞等待回调。

#### 回调队列 （Callback Queue）

一般来说，通过 `RabbitMQ` 执行 `RPC` 是很容易的, 客户端发送请求消息，服务器用响应消息应答。 为了接收响应，我们需要发送一个带有请求的"回调"队列地址。 我们可以使用默认队列。 让我们试一试:
```java
import com.rabbitmq.client.AMQP.BasicProperties;
// ...

callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());

// ... then code to read a response message from the callback_queue ...
```

> 消息属性
> Amqp 0-9-1协议预先定义了一组包含14个属性的消息。 除下列情况外，大多数属性很少使用:
>
> - deliveryMode : 将消息标记为持久化
> - contentType : 用于描述编码的`mime`类型。例如，对于经常使用的 `JSON` 编码，将此属性设置为:`application/json`
> - replyTo: 通常用于命名回调队列
> - correlationId: 将 RPC 响应与请求联系起来

#### 关联ID （Correlation Id）

在上述方法中，我们为每个 `RPC` 请求创建一个回调队列,这非常低效，但幸运的是，有一种更好的方法----让我们为每个客户机创建一个回调队列。

这引发了一个新问题，因为在该队列中接收到了响应，所以不清楚响应属于哪个请求。 而要分辨属于哪个请求就得使用到 `correlationId` 。 我们将为每个请求设置一个惟一`correlationId `。 稍后，当我们在回调队列中收到一条消息时，我们将查看这个属性，并根据这个属性，我们将能够匹配一个响应和一个请求。如果我们看到一个未知的 `correlationId` 值，我们可以安全地丢弃消息——它不属于我们的请求。

您可能会问，为什么我们直接忽略回调队列中的未知消息，而不是因为错误而返回一个失败信息？ 这是因为服务器端可能存在竞态条件。虽然不太可能，但是 RPC 服务器有可能在发送回复给我们之后死掉，但是`ack`并没有发送出来。 如果发生这种情况，重新启动的 RPC 服务器将再次处理请求，这就是为什么在客户机上我们必须优雅地处理重复响应，RPC 理想情况下应该是幂等的。

#### 总结

![summary](http://www.rabbitmq.com/img/tutorials/python-six.png)

我们的 RPC 是这样工作的:
* 对于RPC请求，客户端( `Client` )会发送一个带有两个属性的消息:
  - `replyTo` : 为请求而创建的匿名独占队列名称
  - `correlationId` : 为每一个请求设置的独一无二ID
* 请求会被送到`rpc_queue`中.
* `RPC`持续等待队列中传过来的请求，当队列中消息传到时，`RPC`执行队列消息，并将消息通过`replyTo`中的队列返回
* 客户端等待回复队列上的数据。 出现消息时，它会检查`correlationId`属性。 如果它与请求中的值匹配，则返回对应用程序的响应。

#### 整合一遍

斐波那契数列:
```java
private static int fib(int n) {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```

这是完整的`RPCServer.java`:
```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Envelope;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RPCServer {

  private static final String RPC_QUEUE_NAME = "rpc_queue";

  private static int fib(int n) {
    if (n ==0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
  }

  public static void main(String[] argv) {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");

    Connection connection = null;
    try {
      connection      = factory.newConnection();
      final Channel channel = connection.createChannel();

      channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);
      channel.queuePurge(RPC_QUEUE_NAME);

      channel.basicQos(1);

      System.out.println(" [x] Awaiting RPC requests");

      Consumer consumer = new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
          AMQP.BasicProperties replyProps = new AMQP.BasicProperties
                  .Builder()
                  .correlationId(properties.getCorrelationId())
                  .build();

          String response = "";

          try {
            String message = new String(body,"UTF-8");
            int n = Integer.parseInt(message);

            System.out.println(" [.] fib(" + message + ")");
            response += fib(n);
          }
          catch (RuntimeException e){
            System.out.println(" [.] " + e.toString());
          }
          finally {
            channel.basicPublish( "", properties.getReplyTo(), replyProps, response.getBytes("UTF-8"));
            channel.basicAck(envelope.getDeliveryTag(), false);
            // RabbitMq consumer worker thread notifies the RPC server owner thread 
            synchronized(this) {
            	this.notify();
            }
          }
        }
      };

      channel.basicConsume(RPC_QUEUE_NAME, false, consumer);
      // Wait and be prepared to consume the message from RPC client.
      while (true) {
      	synchronized(consumer) {
      		try {
      			consumer.wait();
      	    } catch (InterruptedException e) {
      	    	e.printStackTrace();	    	
      	    }
      	}
      }
    } catch (IOException | TimeoutException e) {
      e.printStackTrace();
    }
    finally {
      if (connection != null)
        try {
          connection.close();
        } catch (IOException _ignore) {}
    }
  }
}
```

服务器代码相当明了:
- 通常，我们从建立连接、通道和声明队列开始

- 我们可能需要运行多个服务器进程。 为了在多个服务器上平均分配负载，我们需要在`basicQos`中设置`prefetchCount`。
- 使用`basicConsume `连接队列,并提供一个默认回调

这是完整的`RPCClient.java`:
```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Envelope;

import java.io.IOException;
import java.util.UUID;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeoutException;

public class RPCClient {

  private Connection connection;
  private Channel channel;
  private String requestQueueName = "rpc_queue";

  public RPCClient() throws IOException, TimeoutException {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");

    connection = factory.newConnection();
    channel = connection.createChannel();
  }

  public String call(String message) throws IOException, InterruptedException {
    final String corrId = UUID.randomUUID().toString();

    String replyQueueName = channel.queueDeclare().getQueue();
    AMQP.BasicProperties props = new AMQP.BasicProperties
            .Builder()
            .correlationId(corrId)
            .replyTo(replyQueueName)
            .build();

    channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));

    final BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);

    String ctag = channel.basicConsume(replyQueueName, true, new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        if (properties.getCorrelationId().equals(corrId)) {
          response.offer(new String(body, "UTF-8"));
        }
      }
    });

    String result = response.take();
    channel.basicCancel(ctag);
    return result;
  }

  public void close() throws IOException {
    connection.close();
  }

  public static void main(String[] argv) {
    RPCClient fibonacciRpc = null;
    String response = null;
    try {
      fibonacciRpc = new RPCClient();

      for (int i = 0; i < 32; i++) {
        String i_str = Integer.toString(i);
        System.out.println(" [x] Requesting fib(" + i_str + ")");
        response = fibonacciRpc.call(i_str);
        System.out.println(" [.] Got '" + response + "'");
      }
    }
    catch  (IOException | TimeoutException | InterruptedException e) {
      e.printStackTrace();
    }
    finally {
      if (fibonacciRpc!= null) {
        try {
          fibonacciRpc.close();
        }
        catch (IOException _ignore) {}
      }
    }
  }
}
```

客户端代码稍微复杂一些::
- 建立连接、通道和声明队列。
- 提供`run`方法，已提供`RPC`执行。
  * 首先，我们创建一个独一无二的`correlationId`，`RpcConsumer`将会使用这个值来获取响应。
  * 其次，我们创建一个专用于回复信息的队列，然后订阅它。
  * 接下来，发出带有`replyTo`和`correlationId`两个属性的请求消息。
  * 由于我们的消费者交付处理是在一个单独的线程中进行的，因此我们需要在响应到来之前暂停主线程。 使用`BlockingQueue`是一种可能的解决方案。 这里我们创建了`ArrayBlockingQueue`，容量设置为1，因为我们只需要等待一个响应。
  * `handleDelivery`方法正在做一个非常简单的工作。对于每个消耗的响应消息，它检查correlationId是否是我们正在寻找的那个。 如果是这样，它会将响应置于`BlockingQueue`。
  * 同时,主线程正在等待从BlockingQueue获取的响应。最后，我们将响应返回给用户。


客户端创建一个请求:
```java
RPCClient fibonacciRpc = new RPCClient();

System.out.println(" [x] Requesting fib(30)");
String response = fibonacciRpc.call("30");
System.out.println(" [.] Got '" + response + "'");

fibonacciRpc.close();
```

