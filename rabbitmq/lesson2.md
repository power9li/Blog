### Rabbit MQ lesson 2 

[TOC]

### Work Queues(工作队列)


![img](https://www.rabbitmq.com/img/tutorials/python-two.png)

在第一个教程中，我们编写了从指定队列发送和接收消息的程序。在这一项中，我们将创建一个工作队列，用于在多个工作者之间分配耗时的任务。

工作队列的主要思想(即:任务队列)是为了避免立即执行资源密集型任务，并且必须等待它完成。相反，我们将任务安排在稍后完成。我们将任务封装为消息并将其发送到队列中。在后台运行的一个工人进程将会弹出任务并最终执行该任务。当你管理许多员工时，任务就会在他们之间共享。

这个概念在web应用程序中尤其有用，因为在短HTTP请求窗口中不可能处理复杂的任务。

#### 准备

在本教程的前一部分，我们发送了一个包含“Hello World！”的消息。现在，我们将发送支持复杂任务的字符串。我们没有一个真实的任务，比如要缩放的图像或者pdf文件，所以让我们假装我们很忙——通过使用thread.sleep()函数来假装它。我们将把消息字符串里点的数量作为它的复杂度;每一个点都会有一秒钟的“工作”。例如，一个由“Hello… ”所描述的虚假任务。需要三秒钟。

我们将稍微修改一下发送。前一个示例中的java代码，允许从命令行发送任意消息。这个程序将把任务安排到我们的工作队列中，所以让我们把它命名为newtask.java:

```java
String message = getMessage(argv);

channel.basicPublish("", "hello", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```



一些帮助从命令行参数中获取消息:

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

我们的老Recv。java程序还需要进行一些更改:它需要为消息体中的每个点假做一秒钟的工作。它将处理传递的消息并执行任务，因此我们将其命名为Worker.java:

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

模拟执行时间的假任务:

```java
private static void doWork(String task) throws InterruptedException {
    for (char ch: task.toCharArray()) {
        if (ch == '.') Thread.sleep(1000);
    }
} 
```

将它们编译成第1部分(使用工作目录中的jar文件和环境变量CP):

```shell
javac -cp $CP NewTask.java Worker.java
```

####循环调度

使用任务队列的一个优点是能够轻松地并行工作。如果我们正在积累一份积压的工作，我们可以增加更多的工人，这样就可以很容易地扩大规模。

首先，让我们尝试同时运行两个Worker实例。它们都会从队列中获取消息，但具体如何?让我们来看看。

你需要打开三个控制台。两个将运行这个Worker程序。这些控制台将是我们的两个消费者-C1和C2。

```shell
# shell 1
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
```

```shell
# shell 2
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个方面，我们将发布新的任务。一旦你启动了消费者，你就可以发布一些信息:

```shell
# shell 3
java -cp $CP NewTask
# => First message.
java -cp $CP NewTask
# => Second message..
java -cp $CP NewTask
# => Third message...
java -cp $CP NewTask
# => Fourth message....
java -cp $CP NewTask
# => Fifth message.....
```

让我们看看给我们的员工带来了什么:

```shell
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```shell
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ将按顺序将每个消息发送给下一个消费者。平均每个消费者将得到相同数量的消息。这种分发消息的方式称为循环。试着和三个或更多的工人一起试一试。

####消息确认

完成一项任务可能需要几秒钟。你可能会想，如果一个消费者开始一项长时间的任务，并且只完成了一部分，那么会发生什么。在我们当前的代码中，一旦RabbitMQ向客户发送一条消息，它立即将其标记为删除。在这种情况下，如果您杀死了一个工人，我们将丢失它正在处理的消息。我们还将丢失发送给这个特定工作者的所有消息，但是还没有处理。

但我们不想失去任何任务。如果一个工人死了，我们希望这个任务被交付给另一个工人。

为了确保消息不会丢失，RabbitMQ支持消息确认。一个ack([nowledgement](https://www.rabbitmq.com/confirms.html))由使用者返回，告诉RabbitMQ，已经接收到一个特定的消息，并且RabbitMQ可以自由地删除它。

如果一个消费者死亡(它的通道是关闭的，连接是关闭的，或者是TCP连接丢失)，而没有发送ack，那么RabbitMQ将会理解一条消息没有被完全处理，并将重新队列。如果在同一时间线上有其他消费者，那么它将很快地把它重新交付给另一个消费者。这样你就可以确保没有信息丢失，即使工人偶尔会死亡。

没有任何消息超时;RabbitMQ将在用户死亡时重新传递消息。即使处理消息需要很长时间，这也很好。

缺省情况下，手动消息确认将被打开。在前面的例子中，我们通过autoAck=true标记显式地关闭了它们。是时候把这个标志设置为错误，并在完成任务后向员工发送适当的确认信息。

```java
channel.basicQos(1); // 一次只接受一个unack-ed信息的消息(请参见下面的内容)

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

使用这段代码，我们可以确定，即使在处理消息时使用ctrl+c杀死一个工人，也不会丢失任何东西。在工人死后不久，所有未被确认的消息将被重新发送。

#### 被遗忘的ACK

错过basicAck是一个常见的错误。这是一个容易犯的错误，但后果是严重的。当您的客户机退出时，消息将被重新发送(这看起来像是随机的重新发送)，但是RabbitMQ将会消耗更多的内存，因为它将无法释放任何未被释放的消息。

为了调试这类错误，您可以使用rabbitmqctl来打印 messages_unacknowledged 字段:

```shell
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

在Windows上，删除sudo:

```powershell
rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
```

#### 消息的耐久性

我们已经学会了如何确保即使消费者死亡，任务也不会丢失。但是如果RabbitMQ服务器停止，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它将会忘记队列和消息，除非您告诉它不要这样做。需要有两件事来确保消息不会丢失:我们需要将队列和消息标记为持久的。

首先，我们需要确保RabbitMQ永远不会丢失我们的队列。为了实现这一目的，我们需要将其声明为持久的:

```java
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

尽管这个命令本身是正确的，但它在我们当前的设置中是无效的。这是因为我们已经定义了一个名为hello的队列，它不是持久的。RabbitMQ不允许您重新定义具有不同参数的现有队列，并将返回任何试图执行此操作的程序的错误。但是有一个快速的解决方法——让我们声明一个有不同名称的队列，例如taskqueue:

```java
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```

该队列声明更改需要应用于生产者和消费者代码。

此时，我们确信即使RabbitMQ重新启动，任务队列队列也不会丢失。现在，我们需要通过将MessageProperties(实现BasicProperties)设置为值PERSISTENT_TEXT_PLAIN.来标记我们的消息。

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

#### 注意消息的持久性

将消息标记为持久性并不能完全保证消息不会丢失。尽管它告诉RabbitMQ将消息保存到磁盘上，但是当RabbitMQ接受消息并没有保存它时，仍然有一个很短的时间窗口。另外，RabbitMQ不会为每条消息执行fsync(2)——它可能只是保存到缓存中，而不是真正写到磁盘上。持久性保证并不强大，但对于我们的简单任务队列来说，这已经足够了。如果你需要一个更强大的保证，那么你可以使用[出版商的确认](https://www.rabbitmq.com/confirms.html)。

#### 公平的分配

您可能已经注意到，分派仍然不能按照我们的要求工作。例如，在一个有两名员工的情况下，当所有奇怪的消息都很重，甚至消息都很轻时，一个工人就会一直忙碌，而另一个工人几乎不会做任何工作。不过，RabbitMQ对此一无所知，它仍然会将消息平均分配。

这是因为RabbitMQ在消息进入队列时仅发送一条消息。它不考虑消费者的未确认消息的数量。它只是盲目地将每个n个消息发送给第n个消费者。

![img](https://www.rabbitmq.com/img/tutorials/prefetch-count.png)

为了克服这个问题，我们可以使用prefetchCount=1设置的basicQos方法。这就告诉RabbitMQ不要一次给一个工人发送多个消息。或者，换句话说，在处理和承认之前的工作时，不要向工作人员发送一条新消息。相反，它将把它分派给下一个不太忙的工人。

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

**注意队列大小**

***如果所有的工人都很忙，你的队伍就可以填满了。你会想要关注这个问题，可能会增加更多的员工，或者有其他的策略。***

#### 把它放在一起

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

[(NewTask.java source)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/NewTask.java)

And our Worker.java:

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

[(Worker.java source)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Worker.java)

使用消息确认和prefetchCount，您可以设置一个工作队列。即使RabbitMQ重新启动，持久性选项也可以让任务继续存在。

有关通道方法和消息属性的更多信息，您可以在网上浏览[JavaDocs online](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/).。

现在，我们可以继续学习[tutorial 3](https://www.rabbitmq.com/tutorials/tutorial-three-java.html) ，并学习如何向许多消费者传递相同的信息。
