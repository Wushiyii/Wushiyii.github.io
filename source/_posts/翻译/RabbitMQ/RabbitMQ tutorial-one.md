---
title: Rabbit tutorial-one
date: 2018-11-29 19:45:48
tags: [编程,分布式,消息队列]
categories: 消息队列	
---

RabbitMQ入门教程系列---（1）

  <!-- more -->

  # RabbitMQ tutorial-one

  #### RabbitMQ简介

  `RabbitMQ`是一个消息代理程序，它接受和转发消息。 你可以把它想象成一个邮局: 当你把你想要投递的邮件放在一个邮筒里，你可以确定邮递员最终会把邮件投递给你的收件人。 在这个比喻中，`RabbitMQ` 是一个邮筒、一个邮局和一个邮递员。

  `Rabbitmq` 和邮局之间的主要区别在于，它不处理纸张，而是接受、存储和转发二进制blob的数据消息。

  #### 一些术语

  - Producing

    `Producing ` 生产，即为发送消息的程序是一个生产者:

    ![producing](http://www.rabbitmq.com/img/tutorials/producer.png)

  - Queue

    `Queue`队列，队列是在 RabbitMQ 内部的`邮筒`的名称。 尽管消息流经 `RabbitMQ` 和应用程序中，但是它们只能存储在队列中。 队列只受主机的内存和磁盘限制约束，它实际上是一个很大的消息缓冲区。 许多生产者(`Producer`)可以发送到一个队列的消息，许多消费者(`Consumer`)可以从这个队列接收并处理消息。 这就是我们表示队列的方式:

    ![quene](http://www.rabbitmq.com/img/tutorials/queue.png)


  - Consuming
    `Consuming`消费，和接收(`receving`)有着相似的含义。消费者大多数都是一个主要等待接收消息的程序:

    ![consuming](http://www.rabbitmq.com/img/tutorials/consumer.png)

#### "Hello world"

在本教程的这一部分，我们将用 Java 编写两个程序; 一个生产者发送一条消息，另一个消费者接收消息并打印出来。 我们将略过 java api 中的一些细节，主要关注这个非常简单的东西，只是刚刚开始。 这是一个信息的"Hello World"。

在下图中,"p"是生产者,"c"是消费者,中间的框是一个队列—— RabbitMQ 代表使用者保存的消息缓冲区。

![p to c](http://www.rabbitmq.com/img/tutorials/python-one.png)

> Rabbitmq 使用了多种协议。 本教程使用 AMQP 0-9-1，这是一个开放的、通用的消息传递协议。 有很多种语言版本的 RabbitMQ 客户端。 我们将使用 RabbitMQ 提供的 Java 客户端。



- Sending 

  ![sending](http://www.rabbitmq.com/img/tutorials/sending.png)

  我们将称呼消息发布者(`Sender`)` Send` 和称呼消息使用者(`Receiver`)` Recv`。 消息发布者将连接到 `RabbitMQ`，发送一条消息，然后退出。

  在`send.java`中，首先导入一些包（`amqp-client-4.0.2.jar`,`slf4j-api-1.7.21`,`slf4j-simple-1.7.22`）

  ```java
  import com.rabbitmq.client.ConnectionFactory;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.Channel;
  
  public class Send {
      //命名队列
    private final static String QUEUE_NAME = "hello";
  
    public static void main(String[] argv)
        throws java.io.IOException {
        
    }
  }    
  ```

  和服务器建立连接

  ```java
  ConnectionFactory factory = new ConnectionFactory();
  factory.setHost("localhost");
  Connection connection = factory.newConnection();
  Channel channel = connection.createChannel();
  ```

  该连接对`Socket`连接进行抽象，并为我们处理协议版本协商和身份验证等问题。 在这里我们连接到本地机器上的`Broker`。 如果我们想连接到不同机器上的代理，我们只需在这里指定它的名称或 IP 地址。

  接下来创建一个通道`Channel`，这里包括大多数用于完成业务需要 用到的 API。

  为了发送消息，我们必须声明一个队列以供我们发送，然后将消息发布到队列:

  ```java
  channel.queueDeclare(QUEUE_NAME, false, false, false, null);
  String message = "Hello World!";
  channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
  System.out.println(" [x] Sent '" + message + "'");
  ```

  声明队列是幂等的——只有当队列不存在时才创建它。 消息内容是一个字节数组，因此我们可以在那里对任何内容进行编码及修改。

  最后，关闭通道和连接;

  ```java
  channel.close();
  connection.close();
  ```

  点击此处查看完整的`Send.java`类。

- Receiving
消费者是通过 `RabbitMQ` 获取消息，与发布单个消息的发布者不同，我们将让消费者保持监听消息的状态，并打印出消息。
![rec](http://www.rabbitmq.com/img/tutorials/receiving.png)
导入包
```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
```
`DefaultConsumer`接口实现了`Consumer`接口，我们将使用该接口来缓冲服务器推送给我们的消息。
设置与发布者(`Sender`)相同; 我们打开一个连接和一个通道，并声明我们要使用的队列。 请注意，这与发布者的队列名称相同。
```java
public class Recv {
  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv)
      throws java.io.IOException,
             java.lang.InterruptedException {

    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
    ...
    }
}
```
注意，我们在这里同样也声明了队列，因为我们可能在发布者之前启动使用者，所以我们希望在尝试使用来自该队列的消息之前确保队列的存在。

接下来我们将要告诉发布者从队列中传递消息，因为它将以异步方式推送我们的消息。接下来使用`DefaultConsumer`来缓冲消息，直到我们准备好使用它们。
```java
Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope,
                             AMQP.BasicProperties properties, byte[] body)
      throws IOException {
    String message = new String(body, "UTF-8");
    System.out.println(" [x] Received '" + message + "'");
  }
};
channel.basicConsume(QUEUE_NAME, true, consumer);
```
  点击此处查看完整的`Recv.java`类。


#### 将两个实例同时运行
你可以在类路径中通过 RabbitMQ java 客户端编译这两个代码:
```bash
javac -cp amqp-client-4.0.2.jar Send.java Recv.java
```
要运行它们，还需要`rabbitmq-client.jar`及其依赖。在终端中，运行消费者(`Consumer`) :
```bash
java -cp .:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar Recv
```
然后，运行发布者(`Producer`) :
```bash
java -cp .:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar Send
```
> 在 `Windows` 上，使用**分号**而不是**冒号**分隔类路径中的项。

消费者将通过` RabbitMQ` 打印从发布者那里获得的消息。 使用者将继续运行，等待消息(使用 Ctrl-C 停止它) ，因此尝试从另一个终端运行发布者。
> 你可能希望看到`RabbitMQ` 运行着什么队列以及队列中有多少消息。你可以(管理以模式)使用` rabbitmqctl `工具:
```bash
sudo rabbitmqctl list_queues
```
或者在`Windows`上省略sudo:
```bash
rabbitmqctl.bat list_queues
```

#### Tips
为了保存输入，你可以为路径设置一个环境变量:
```bash
export CP=.:amqp-client-4.0.2.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.22.jar
java -cp $CP Send
```
或者在`Windows`上:
```bash
set CP=.;amqp-client-4.0.2.jar;slf4j-api-1.7.21.jar;slf4j-simple-1.7.22.jar
java -cp %CP% Send
```
> 本教程全文翻译自 `http://www.rabbitmq.com/tutorials/tutorial-one-java.html`