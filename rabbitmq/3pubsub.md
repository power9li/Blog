## RabbiMQ Tutorial3 Publish/Subscribe

[TOC]

### Publish/Subscribe(发布与订阅)

在 [前一教程 ](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)我们创建了一个工作 队列中。 工作队列背后的假设是,每个任务 交付给一个工人。 在这一部分我们将做些完全不同的事,我们将把消息发送给多个消费者。 这种模式被称为“发布/订阅”。

为了说明这个模式,我们将构建一个简单的日志记录 系统。 它将包括两个项目,第一个将发出日志 消息,第二个将接收并打印。

在我们的日志系统每个接收机的副本程序都将收到消息。 这样我们就能运行一个接收器 直接把日志写到磁盘上; 同时我们将能够运行另一个接收器,把日志显示到屏幕上。

本质上,发表日志消息将被广播给所有 接收器。

####转换【Exchange】

在之教程的之前部分我们从一个队列发送和接收消息。现在是时候介绍RabbitMQ的完整消息模型了 。

让我们快速回顾我们在前面的教程:

- 一个 *producer*是一个用户应用程序发送消息。
- 一个 *queue* 是一个缓冲区,它存储信息。
- 一个 *consumer*是一个用户应用程序接收消息。

RabbitMQ的消息模型的核心思想是*producer*从来没有直接向*queue*发送任何消息。 实际上,*producer*通常甚至根本不知道消息将传递给哪个*queue*。

相反,*producer*只能发送消息到一个 *Exchange*。 一个 *Exchange*是一个非常简单的东西。 从*producer*一侧接收消息 并且另一边把消息推送给*queue*。 *Exchange* 必须确切的清楚如何处理接收的消息。 是应该把它推送到一个特定的*queue*吗? 还是应该是追加到多个*queue*? 或者应该丢弃。 这些该如何做的规则，定义在 *Exchange type* 中。

![img](https://www.rabbitmq.com/img/tutorials/exchanges.png)

这有一些可用的Echange type:```direct, topic, headers``` 和 ```fanout``` 我们将首先关注最后一个 —> *fanout* .(扇出)

让我们创建一个 Exchange 类型 = fanout 的Exchange, 称它为logs:

```java
channel.exchangeDeclare("logs", "fanout");
```

fanout Exchange 非常简单，你大概已经从它的名字猜出来了，它仅仅广播所有消息到它知道的所有queue里。但这正式我们日志记录器所需要的。



> **交换列表 【Listing Exchange】**
>
> 要列出Server上的 Exchange ，需要使用```rabbitmqctl``` 命令
>
> ```shell
> sudo rabbitmqctl list_exchanges
> ```
>
> 列表中会有一些 amp.* 的 Exchange 和默认的 Exchange (没有名字的)。他们是默认就被创建出来的，不过目前你不太可能会用得到它们。
>
> **没有名字的 Exchange**
>
> 在之前的教程中，我们不知道什么是 Exchange, 但是我们仍然把消息发送到了queue。这是因为它们使用了默认的空字符串String("") 的 Exchange。
>
> 回忆我们之前发送的消息：
>
> ```java
> channel.basicPublish("", "hello", null, message.getBytes());
> ```
>
> 第一个参数是 exchange的名字。空字符串代表使用默认的没有名字的 exchange; 消息将被路由到指定名称的队列中，如果它存在的话。



现在，让我们用我们有名字的 exchange来替换：

```java
channel.basicPublish( "logs", "", null, message.getBytes());
```



#### 临时队列【Temporary queue】

你可能记得以前我们使用队列了 指定的名称(还记得hello和 task_queue吗？) 一个有名字的queue对我们是重要的—> 我们需要制定发送者发送到相同的queue。**当你希望在producer 和 consumer之间共享队列的时候，使用一个相同名字的队列名非常重要**。

但是它并不适用与我们的日志记录器（logger)。我们将监听所有的消息,而不是一部分。我们也只对当前的消息感兴趣，而不是旧的消息。要解决这个问题，我们需要两件事。

首先，当我们与RabbitMQ连接时，我们需要一个新鲜的、空的队列。要做到这一点，我们可以创建一个带有随机名称的队列，或者，甚至更好——让服务器为我们选择一个随机的队列名称。

其次，一旦我们断开了消费者的连接，队列就会自动被删除。

在Java客户机中，当我们调用不带参数的queueDeclare()方法时，将创建了一个非持久的、排他的、自动删除队列，该队列有一个生成的名称:

```java
String queueName = channel.queueDeclare().getQueue();
```

这时，queueName包含一个随机的队列名。例如，它可能看起来像   ```amq.gen-JzTY20BRgKO-HjmUJj0wLg.```

####绑定【Bindings】

![img](https://www.rabbitmq.com/img/tutorials/bindings.png)



我们已经创建了一个fanout交换器和一个队列。现在我们需要告诉交换器将消息发送到我们的队列。交换和队列之间的关系称为绑定。

```java
channel.queueBind(queueName, "logs", "");
```

从现在开始，日志Exchange将向我们的队列追加消息。

> #### Listing bindings
>
> You can list existing bindings using, you guessed it,
>
> ```shell
> rabbitmqctl list_bindings
> ```

#### Putting it all together(把上面的合并)

![img](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)



生成日志消息的生产者程序与前面的教程没有什么不同。最重要的变化是,我们将发布消息到我们的 *log exchange* 而不是之前的默认匿名 exchange。我们需要在发送时提供一个路由键*routingKey*，但是在s会用fanout Exchange时，它的值被忽略了。这里是*EmitLog.java*:

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

[EmitLog.java source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLog.java)



如您所见，在建立连接后，我们声明了交换。这一步是必要的，因为发布到一个非现有的交换是被禁止的。

如果没有队列绑定到交换器，消息将丢失，但这对我们来说是可以的;如果没有消费者在听，我们就可以安全地丢弃这个消息。

The code for ReceiveLogs.java:

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

[ReceiveLogs.java source](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogs.java)

像之前一样，进行编译

```shell
javac -cp $CP EmitLog.java ReceiveLogs.java
```

如果您想要将日志保存到文件中，只需打开一个控制台并输入:

```shell
java -cp $CP ReceiveLogs > logs_from_rabbit.log
```

如果您希望看到屏幕上的日志，生成一个新的终端并运行:

```shell
java -cp $CP ReceiveLogs
```

当然，要发送日志:

```shell
java -cp $CP EmitLog
```

使用 **rabbitmqctl list_bindings** 命令，您可以验证代码实际上是根据我们需要创建绑定和队列的。有两个ReceiveLogs。运行您的java程序应该会看到类似的东西:

```shell
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

对结果的解释很简单:来自交换日志的数据将被分配到带有服务器分配名称的两个队列。这正是我们想要的。

要了解如何侦听消息的子集，让我们继续学习 [第4部分](https://www.rabbitmq.com/tutorials/tutorial-four-java.html)



============================

本地试验

```SHELL
cd /Users/shenli/develop/testcodes
```

启动第一个接收器

```shell
java -jar ReceiveLogs.jar >logs_from_rabbit.log
```

启动第二个接收器

```shell
java -jar ReceiveLogs.jar 
 [*] Waiting for messages. To exit press CTRL+C
```

查看exchange：

```shell
/usr/local/Cellar/rabbitmq/3.7.2/sbin/rabbitmqctl list_bindings|grep logs
logs	exchange	amq.gen--Q4vmLWHmazji1lte9SXqA	queue		[]
logs	exchange	amq.gen-oGfYSnvEr6ASjIZNnCDK2g	queue		[]
```

发送消息：

```shell
shenli@lideMacBook-Pro testcodes$ java -jar EmitLog.jar aaaa
 [x] Sent 'aaaa'
shenli@lideMacBook-Pro testcodes$ java -jar EmitLog.jar bbbb
 [x] Sent 'bbbb'
shenli@lideMacBook-Pro testcodes$ java -jar EmitLog.jar cccc
 [x] Sent 'cccc'
shenli@lideMacBook-Pro testcodes$
```

查看消息接收：

1控制台

```shell
Cshenli@lideMacBook-Pro testcodes$ java -jar ReceiveLogs.jar 
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'aaaa'
 [x] Received 'bbbb'
 [x] Received 'cccc'
```

2 日志

```shell
cat logs_from_rabbit.log 
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'aaaa'
 [x] Received 'bbbb'
 [x] Received 'cccc'
```

试验成功，说明 **fanout**方式，直接将消息发送给了所有binding到该Exchagne上的队列。

这种应用场景应该是所有接收端可以幂等的处理消息的情况。
