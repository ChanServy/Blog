---
title: RabbitMQ学习记录
date: 2022-06-22 22:08:12
categories: 消息队列
tags: RabbitMQ
urlname: rabbitmq-study
---

# RabbitMQ-基础篇

## 初识MQ

### 同步和异步通讯

微服务间通讯有同步和异步两种方式：

同步通讯：就像打电话，需要实时响应。

异步通讯：就像发邮件，不需要马上回复。

![image-20210717161939695](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717161939695.png)

两种方式各有优劣，打电话可以立即得到响应，但是你却不能跟多个人同时通话。发送邮件可以同时与多个人收发邮件，但是往往响应会有延迟。

<!--more-->

#### 同步通讯

我们熟知的微服务间通信Feign调用就属于同步方式，虽然调用可以实时得到结果，但存在下面的问题：

![image-20210717162004285](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717162004285.png)



总结：

同步调用的优点：

- 时效性较强，可以立即得到结果

同步调用的问题：

- 耦合度高
- 性能和吞吐能力下降
- 有额外的资源消耗
- 有级联失败问题



#### 异步通讯

异步调用则可以避免上述问题：



我们以购买商品为例，用户支付后需要调用订单服务完成订单状态修改，调用物流服务，从仓库分配响应的库存并准备发货。

在事件模式中，支付服务是事件发布者（publisher），在支付完成后只需要发布一个支付成功的事件（event），事件中带上订单id。

订单服务和物流服务是事件订阅者（Consumer），订阅支付成功的事件，监听到事件后完成自己业务即可。



为了解除事件发布者与订阅者之间的耦合，两者并不是直接通信，而是有一个中间人（Broker）。发布者发布事件到Broker，不关心谁来订阅事件。订阅者从Broker订阅事件，不关心谁发来的消息。

![image-20210422095356088](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210422095356088.png)



Broker 是一个像数据总线一样的东西，所有的服务要接收数据和发送数据都发到这个总线上，这个总线就像协议一样，让服务间的通讯变得标准和可控。



好处：

- 吞吐量提升：无需等待订阅者处理完成，响应更快速

- 故障隔离：服务没有直接调用，不存在级联失败问题
- 调用间没有阻塞，不会造成无效的资源占用
- 耦合度极低，每个服务都可以灵活插拔，可替换
- 流量削峰：不管发布事件的流量波动多大，都由Broker接收，订阅者可以按照自己的速度去处理事件



缺点：

- 架构复杂了，业务没有明显的流程线，不好管理
- 需要依赖于Broker的可靠、安全、性能





好在现在开源软件或云平台上 Broker 的软件是非常成熟的，比较常见的一种就是我们今天要学习的MQ技术。



### 技术对比

MQ，中文是消息队列（MessageQueue），字面来看就是存放消息的队列。也就是事件驱动架构中的Broker。

比较常见的MQ实现：

- ActiveMQ
- RabbitMQ
- RocketMQ
- Kafka



几种常见MQ的对比：

|            | **RabbitMQ**            | **ActiveMQ**                   | **RocketMQ** | **Kafka**  |
| ---------- | ----------------------- | ------------------------------ | ------------ | ---------- |
| 公司/社区  | Rabbit                  | Apache                         | 阿里         | Apache     |
| 开发语言   | Erlang                  | Java                           | Java         | Scala&Java |
| 协议支持   | AMQP，XMPP，SMTP，STOMP | OpenWire,STOMP，REST,XMPP,AMQP | 自定义协议   | 自定义协议 |
| 可用性     | 高                      | 一般                           | 高           | 高         |
| 单机吞吐量 | 一般                    | 差                             | 高           | 非常高     |
| 消息延迟   | 微秒级                  | 毫秒级                         | 毫秒级       | 毫秒以内   |
| 消息可靠性 | 高                      | 一般                           | 高           | 一般       |

追求可用性：Kafka、 RocketMQ 、RabbitMQ

追求可靠性：RabbitMQ、RocketMQ

追求吞吐能力：RocketMQ、Kafka

追求消息低延迟：RabbitMQ、Kafka



## 快速入门

### 安装RabbitMQ

https://chanservy.vercel.app/posts/20220622/rabbitmq-install.html

MQ的基本结构：

![image-20210717162752376](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717162752376.png)



RabbitMQ中的一些角色：

- publisher：生产者
- consumer：消费者
- exchange：交换机，负责消息路由
- queue：队列，存储消息
- virtualHost：虚拟主机，隔离不同租户的exchange、queue、消息的隔离





### RabbitMQ消息模型

RabbitMQ官方提供了5个不同的Demo示例，对应了不同的消息模型：

![image-20210717163332646](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717163332646.png)







### Demo案例

案例代码地址：https://github.com/ChanServy/rabbitmq-advanced

包括三部分：

- mq-demo：父工程，管理项目依赖
- publisher：消息的发送者
- consumer：消息的消费者
- common：公共模块



### 入门案例

简单队列模式的模型图：

 ![image-20210717163434647](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717163434647.png)

官方的HelloWorld是基于最基础的消息队列模型来实现的，只包括三个角色：

- publisher：消息发布者，将消息发送到队列queue
- queue：消息队列，负责接受并缓存消息
- consumer：订阅队列，处理队列中的消息





#### publisher实现

思路：

- 建立连接
- 创建Channel
- 声明队列
- 发送消息
- 关闭连接和channel



代码实现：

```java
package cn.itcast.mq.helloworld;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import org.junit.Test;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class PublisherTest {
    @Test
    public void testSendMessage() throws IOException, TimeoutException {
        // 1.建立连接
        ConnectionFactory factory = new ConnectionFactory();
        // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
        factory.setHost("192.168.150.101");
        factory.setPort(5672);
        factory.setVirtualHost("/");
        factory.setUsername("itcast");
        factory.setPassword("123321");
        // 1.2.建立连接
        Connection connection = factory.newConnection();

        // 2.创建通道Channel
        Channel channel = connection.createChannel();

        // 3.创建队列
        String queueName = "simple.queue";
        channel.queueDeclare(queueName, false, false, false, null);

        // 4.发送消息
        String message = "hello, rabbitmq!";
        channel.basicPublish("", queueName, null, message.getBytes());
        System.out.println("发送消息成功：【" + message + "】");

        // 5.关闭通道和连接
        channel.close();
        connection.close();

    }
}
```







#### consumer实现

代码思路：

- 建立连接
- 创建Channel
- 声明队列
- 订阅消息



代码实现：

```java
package cn.itcast.mq.helloworld;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class ConsumerTest {

    public static void main(String[] args) throws IOException, TimeoutException {
        // 1.建立连接
        ConnectionFactory factory = new ConnectionFactory();
        // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
        factory.setHost("192.168.150.101");
        factory.setPort(5672);
        factory.setVirtualHost("/");
        factory.setUsername("itcast");
        factory.setPassword("123321");
        // 1.2.建立连接
        Connection connection = factory.newConnection();

        // 2.创建通道Channel
        Channel channel = connection.createChannel();

        // 3.创建队列
        String queueName = "simple.queue";
        channel.queueDeclare(queueName, false, false, false, null);

        // 4.订阅消息
        channel.basicConsume(queueName, true, new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                // 5.处理消息
                String message = new String(body);
                System.out.println("接收到消息：【" + message + "】");
            }
        });
        System.out.println("等待接收消息。。。。");
    }
}
```





### 总结

基本消息队列的消息发送流程：

1. 建立connection

2. 创建channel

3. 利用channel声明队列

4. 利用channel向队列发送消息

基本消息队列的消息接收流程：

1. 建立connection

2. 创建channel

3. 利用channel声明队列

4. 定义consumer的消费行为handleDelivery()

5. 利用channel将消费者与队列绑定





## SpringAMQP

SpringAMQP是基于RabbitMQ封装的一套模板，并且还利用SpringBoot对其实现了自动装配，使用起来非常方便。

SpringAmqp的官方地址：https://spring.io/projects/spring-amqp

![image-20210717164024967](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717164024967.png)

![image-20210717164038678](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717164038678.png)



SpringAMQP提供了三个功能：

- 自动声明队列、交换机及其绑定关系
- 基于注解的监听器模式，异步接收消息
- 封装了RabbitTemplate工具，用于发送消息 



### Basic Queue 简单队列模型

在父工程中引入依赖

```xml
<!--AMQP依赖，包含RabbitMQ-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```



#### 消息发送

首先配置MQ地址，在publisher服务的application.yml中添加配置：

```yaml
spring:
  rabbitmq:
    host: localhost # 主机名
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: root # 用户名
    password: 123456 # 密码
```



然后在publisher服务中编写测试类SpringAmqpTest，并利用RabbitTemplate实现消息发送：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringAmqpTest {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * basic queue 简单队列模型
     * 利用RabbitTemplate实现向指定队列中发送消息。
     * 一个生产者向队列中推送消息，一个消费者监听队列消费消息。
     */
    @Test
    public void testSimpleQueue() {
        // 队列名称
        String queueName = "simple.queue";
        // 消息
        String message = "hello, spring amqp!";
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}

@RestController
@Slf4j
public class MqController {

    @Resource
    RabbitTemplate rabbitTemplate;

    @RequestMapping("/simple")
    public R simpleSendToQueue() {
        Book book = new Book();
        book.setName("蛤蟆先生去看心理医生");
        book.setAuthor("罗伯特");
        String bookJson = JSONUtil.toJsonStr(book);
        rabbitTemplate.convertAndSend("simple.queue", bookJson);
        return R.ok();
    }
}
```





#### 消息接收

首先配置MQ地址，在consumer服务的application.yml中添加配置：

```yaml
spring:
  rabbitmq:
    host: localhost # 主机名
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: root # 用户名
    password: 123456 # 密码
```



新建一个类SimpleQueueListener，代码如下：

注：@RabbitListener注解监听的队列在MQ服务器中如果不存在，会报错。

发送消息，如果发送的消息是个对象，我们会使用序列化机制将对象写出去，对象必须实现 Serializable。

```java
/**
 * 生产者消费者服务间传递的消息是对象时，对象的引用路径必须一致
 * 这样就得将这个对象抽取出来放到公共模块
 * 如果不这样，那么在发送消息之前，生产者可以将这个对象变成JSON字符串
 * 然后再发送，那么在消费者收到消息之后，需要将JSON字符串再转成对象。
 *
 * @author CHAN
 * @since 2022/7/7
 */
@Data
public class Book implements Serializable {
    private String name;
    private String author;
}

```



```java
package com.chan.mq.consumer.listener;

import com.chan.mq.common.pojo.Book;
import com.chan.mq.common.pojo.Movie;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * 监听消息：使用@RabbitListener
 * @RabbitListener ：加在类或者方法上（监听哪些队列即可）queues：声明需要监听的所有队列。
 * @RabbitHandler ：加在方法上（区分不同类型的消息）
 * 区分的不同类型消息：
 * 可能是listener监听两个队列的消息，两个队列中的消息类型可能不同，String、Book。
 * 也可能是listener监听一个队列中的消息，发送到这个队列的多条消息类型可能不同，String、Book。
 *
 * @author CHAN
 * @since 2022/7/7
 */
// @RabbitListener(queues = {"simple.queue", "work.queue"})
@Component
@RabbitListener(queues = {"simple.queue"})
public class SimpleQueueListener {
    /**
     * 监听simple.queue队列中的消息，如果simple.queue不存在会报错。
     * // @RabbitListener(queues = "simple.queue")
     * // public void listenSimpleQueueMessage(String msg) throws InterruptedException {
     * //     System.out.println("spring 消费者接收到消息：【" + msg + "】");
     * // }
     */

    @RabbitHandler
    public void listenSimpleQueueMessage(String msg) {
        System.out.println("spring 消费者接收到String类型的消息：【" + msg + "】");
    }

    @RabbitHandler
    public void listenSimpleQueueMessage(Book book) {
        System.out.println("spring 消费者接收到Book类型的消息：【" + book + "】");
    }
}
```

 * @RabbitListener ：加在类或者方法上，queues：声明需要监听的所有队列。
 * @RabbitHandler ：加在方法上（区分不同类型的消息）
 * 区分的不同类型消息：
   - 可能是listener监听两个队列的消息，两个队列中的消息类型可能不同，String、Book。
   - 也可能是listener监听一个队列中的消息，发送到这个队列的多条消息类型可能不同，String、Book。

- 参数可以写以下类型：
  - 1、Message message：原生消息详细信息。头 + 体
  - 2、T<发送的消息的类型> ：例如 Book
  - 3、Channel channel：当前传输数据的通道。注：如果消费者需要手动 ACK，那么就得传 channel，channel 可以调用相关 API。

> Queue：可以多个消费者都来监听，只要消息被收到，队列删除消息，而且在正常情况下，同一条消息只能有一个消费者收到。
>
> 场景：
>
> - 订单服务启动多个，但是对于同一条消息只能有一个客户端收到。
> - 只有一个消息完全处理完，方法运行结束，我们才可以接收到下一条消息。



#### 测试

启动consumer服务，然后在publisher服务中运行测试代码，发送MQ消息。

> 注意：
> 
> RabbitMQ一共五种消息队列。基本消息队列和工作消息队列的情况下，生产者是直接向队列中发送消息，消费者监听队列就可以。创建队列的代码写在消费者微服务里面，生产者服务只需要知道队列的名字，发送消息时指定队列名和消息就可以。创建队列的方式我们使用@Bean的方式。
> 
> 在广播（Fanout）、路由（Direct）、主题（Topic）的情况下，生产者是向交换机（Exchange）中发送消息，然后绑定队列和交换机，绑定之后交换机将消息传递给队列，消费者依然是监听队列。创建交换机、队列以及绑定的代码写在消费者微服务里面，生产者服务需要知道交换机的名字、交换机和队列间传递消息的routing key，发送消息时指定交换机名和key就可以。创建交换机、队列以及绑定的代码我们可以使用@Bean的方式，也可以直接在@RabbitListener注解中声明交换机并指定交换机类型、队列以及绑定关系。



### WorkQueue

Work queues，也被称为（Task queues），任务模型。简单来说就是**让多个消费者绑定到一个队列，共同消费队列中的消息**。

![image-20210717164238910](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717164238910.png)

当消息处理比较耗时的时候，可能生产消息的速度会远远大于消息的消费速度。长此以往，消息就会堆积越来越多，无法及时处理。

此时就可以使用work 模型，多个消费者共同处理消息处理，速度就能大大提高了。



#### 消息发送

这次我们循环发送，模拟大量消息堆积现象。

在publisher服务中的SpringAmqpTest类中添加一个测试方法：

```java
/**
 * WorkQueues，也被称为（Task queues），任务模型
 * 向队列中不停发送消息，模拟消息堆积。
 * 一个生产者向队列中推送消息，让多个消费者绑定到一个队列，共同消费队列中的消息。避免消息堆积。
 */
@Test
public void testWorkQueue() throws InterruptedException {
    // 队列名称
    String queueName = "work.queue";
    // 消息
    String message = "hello, message_";
    for (int i = 0; i < 10; i++) {
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message + i);
        Thread.sleep(20);
    }
}
```





#### 消息接收

要模拟多个消费者绑定同一个队列，我们在consumer服务的WorkQueueListener中添加2个新的方法：

```java
@Component
public class WorkQueueListener {
    @RabbitListener(queues = "work.queue")
    public void listenWorkQueue1(String msg) throws InterruptedException {
        System.out.println("消费者1接收到消息：【" + msg + "】" + LocalTime.now());
        Thread.sleep(20);
    }

    @RabbitListener(queues = "work.queue")
    public void listenWorkQueue2(String msg) throws InterruptedException {
        System.err.println("消费者2......接收到消息：【" + msg + "】" + LocalTime.now());
        Thread.sleep(200);
    }
}
```

注意到这个消费者sleep了200毫秒，模拟任务耗时，两个消费者任务耗时不一致。





#### 测试

启动ConsumerApplication后，在执行publisher服务中刚刚编写的发送测试方法testWorkQueue。

可以看到消费者1很快完成了自己的25条消息。消费者2却在缓慢的处理自己的25条消息。



也就是说消息是平均分配给每个消费者，并没有考虑到消费者的处理能力。这样显然是有问题的。





#### 能者多劳

在spring中有一个简单的配置，可以解决这个问题。我们修改consumer服务的application.yml文件，添加配置：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 # 每次只能获取一条消息，处理完成才能获取下一个消息
```



#### 总结

Work模型的使用：

- 多个消费者绑定到一个队列，同一条消息只会被一个消费者处理
- 通过设置prefetch来控制消费者预取的消息数量





### 发布/订阅

发布订阅的模型如图：

![image-20210717165309625](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717165309625.png)



可以看到，在订阅模型中，多了一个exchange角色，而且过程略有变化：

- Publisher：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给交换机
- Exchange：交换机。一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于Exchange的类型。Exchange有以下3种类型：
  - Fanout：广播，将消息交给所有绑定到交换机的队列
  - Direct：定向，把消息交给符合指定routing key 的队列
  - Topic：通配符，把消息交给符合routing pattern（路由模式） 的队列
- Consumer：消费者，与以前一样，订阅队列，没有变化
- Queue：消息队列也与以前一样，接收消息、缓存消息。



**Exchange（交换机）只负责转发消息，不具备存储消息的能力**，因此如果没有任何队列与Exchange绑定，或者没有符合路由规则的队列，那么消息会丢失！



### Fanout

Fanout，英文翻译是扇出，我觉得在MQ中叫广播更合适。

![image-20210717165438225](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717165438225.png)

在广播模式下，消息发送流程是这样的：

- 1）  可以有多个队列
- 2）  每个队列都要绑定到Exchange（交换机）
- 3）  生产者发送的消息，只能发送到交换机，交换机来决定要发给哪个队列，生产者无法决定
- 4）  交换机把消息发送给绑定过的所有队列
- 5）  订阅队列的消费者都能拿到消息



我们的计划是这样的：

- 创建一个交换机 itcast.fanout，类型是Fanout
- 创建两个队列fanout.queue1和fanout.queue2，绑定到交换机itcast.fanout

![image-20210717165509466](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717165509466.png)





#### 声明队列和交换机

Spring提供了一个接口Exchange，来表示所有不同类型的交换机：

![image-20210717165552676](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717165552676.png)



在consumer中创建一个类，声明队列和交换机：

```java
/**
 * @author CHAN
 * @since 2022/7/8
 */
@Configuration
public class FanoutConfig {

    /**
     * 声明交换机，默认情况下，由SpringAMQP声明的交换机都是持久化的。
     * @return Fanout类型交换机
     */
    @Bean
    public FanoutExchange fanoutExchange(){
        // 三个参数：交换机名称、是否持久化、当没有queue与其绑定时是否自动删除
        return new FanoutExchange("fanout.exchange", true, false);
    }

    /**
     * 默认情况下，由SpringAMQP声明的队列都是持久化的。
     * 第1个队列
     */
    @Bean
    public Queue fanoutQueue1(){
        return new Queue("fanout.queue1");
        // 使用QueueBuilder构建队列，durable就是持久化的
        // return QueueBuilder.durable("fanout.queue1").build();
    }


    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue1(){
        return BindingBuilder.bind(fanoutQueue1()).to(fanoutExchange());
    }

    /**
     * 第2个队列
     */
    @Bean
    public Queue fanoutQueue2(){
        return new Queue("fanout.queue2");
    }

    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue2(){
        return BindingBuilder.bind(fanoutQueue2()).to(fanoutExchange());
    }

}

```



#### 消息发送

```java
@RequestMapping("/fanout")
public R fanoutSendToExchange() {
    // 交换机名称
    String exchangeName = "fanout.exchange";
    // 消息
    String message = "hello, Rabbit FanoutExchange!";
    rabbitTemplate.convertAndSend(exchangeName, "", message);
    return R.ok();
}
```



#### 消息接收

在consumer服务的WithExchangeListener中添加两个方法，作为消费者：

```java
/**
 * @author CHAN
 * @since 2022/7/8
 */
@Component
public class WithExchangeListener {
    @RabbitListener(queues = "fanout.queue1")
    public void listenFanoutQueue1(String msg) {
        int i = 1 / 0;// test error msg
        System.out.println("消费者1接收到Fanout消息：【" + msg + "】");
    }

    @RabbitListener(queues = "fanout.queue2")
    public void listenFanoutQueue2(String msg) {
        System.out.println("消费者2接收到Fanout消息：【" + msg + "】");
    }
}
```



#### 总结



交换机的作用是什么？

- 接收publisher发送的消息
- 将消息按照规则路由到与之绑定的队列
- 不能缓存消息，路由失败，消息丢失
- FanoutExchange的会将消息路由到每个绑定的队列

声明队列、交换机、绑定关系的Bean是什么？

- Queue
- FanoutExchange
- Binding



### Direct

在Fanout模式中，一条消息，会被所有订阅的队列都消费。但是，在某些场景下，我们希望不同的消息被不同的队列消费。这时就要用到Direct类型的Exchange。

![image-20210717170041447](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717170041447.png)

 在Direct模型下：

- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个`RoutingKey`（路由key）
- 消息的发送方在 向 Exchange发送消息时，也必须指定消息的 `RoutingKey`。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的`Routing Key`进行判断，只有队列的`Routingkey`与消息的 `Routing key`完全一致，才会接收到消息





**案例需求如下**：

1. 利用@RabbitListener声明Exchange、Queue、RoutingKey

2. 在consumer服务中，编写两个消费者方法，分别监听direct.queue1和direct.queue2

3. 在publisher中编写测试方法，向itcast. direct发送消息

![image-20210717170223317](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717170223317.png)





#### 基于注解声明队列和交换机

基于@Bean的方式声明队列和交换机比较麻烦，Spring还提供了基于注解方式来声明。

在consumer的WithExchangeListener中添加两个消费者，同时基于注解来声明队列和交换机：

```java
// 上面的fanout类型的交换机测试的时候我们是使用@Bean的方式创建的交换机和队列以及绑定关系，下面使用注解的方式
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue1"),
    exchange = @Exchange(name = "direct.exchange", type = ExchangeTypes.DIRECT/*交换机类型默认DIRECT*/),
    key = {"red", "blue"}
))
public void listenDirectQueue1(String msg) {
    System.out.println("消费者接收到direct.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue2"),
    exchange = @Exchange(name = "direct.exchange", type = ExchangeTypes.DIRECT),
    key = {"red", "yellow"}
))
public void listenDirectQueue2(String msg) {
    System.out.println("消费者接收到direct.queue2的消息：【" + msg + "】");
}
```



#### 消息发送

在publisher服务中添加测试方法：

```java
@RequestMapping("/direct")
public R directSendToExchange() {
    // 交换机名称
    String exchangeName = "direct.exchange";
    // 消息
    String message = "红色警报！日本乱排核废水，导致海洋生物变异，惊现哥斯拉！";
    // 发送消息
    rabbitTemplate.convertAndSend(exchangeName, "blue", message);
    return R.ok();
}
```





#### 总结

描述下Direct交换机与Fanout交换机的差异？

- Fanout交换机将消息路由给每一个与之绑定的队列
- Direct交换机根据RoutingKey判断路由给哪个队列
- 如果多个队列具有相同的RoutingKey，则与Fanout功能类似

基于@RabbitListener注解声明队列和交换机有哪些常见注解？

- @Queue
- @Exchange





### Topic



#### 说明

`Topic`类型的`Exchange`与`Direct`相比，都是可以根据`RoutingKey`把消息路由到不同的队列。只不过`Topic`类型`Exchange`可以让队列在绑定`Routing key` 的时候使用通配符！



`Routingkey` 一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如： `item.insert`

 通配符规则：

`#`：匹配一个或多个词

`*`：匹配不多不少恰好1个词



举例：

`item.#`：能够匹配`item.spu.insert` 或者 `item.spu`

`item.*`：只能匹配`item.spu`

​     

图示：

 ![image-20210717170705380](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717170705380.png)

解释：

- Queue1：绑定的是`china.#` ，因此凡是以 `china.`开头的`routing key` 都会被匹配到。包括china.news和china.weather
- Queue4：绑定的是`#.news` ，因此凡是以 `.news`结尾的 `routing key` 都会被匹配。包括china.news和japan.news



案例需求：

实现思路如下：

1. 利用@RabbitListener声明Exchange、Queue、RoutingKey

2. 在consumer服务中，编写两个消费者方法，分别监听topic.queue1和topic.queue2

3. 在publisher中编写测试方法，向itcast. topic发送消息



![image-20210717170829229](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210717170829229.png)





#### 消息发送

在publisher服务中添加测试方法：

```java
@RequestMapping("/topic")
public R topicSendToExchange() {
    // 交换机名称
    String exchangeName = "topic.exchange";
    // 消息
    String message = "喜报！孙悟空大战哥斯拉，胜!";
    // 发送消息
    rabbitTemplate.convertAndSend(exchangeName, "china.news", message);
    return R.ok();
}
```



#### 消息接收

在consumer服务的WithExchangeListener中添加方法：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "topic.queue1"),
    exchange = @Exchange(name = "topic.exchange", type = ExchangeTypes.TOPIC),
    key = "china.#"
))
public void listenTopicQueue1(String msg) {
    System.out.println("消费者接收到topic.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "topic.queue2"),
    exchange = @Exchange(name = "topic.exchange", type = ExchangeTypes.TOPIC),
    key = "#.news"
))
public void listenTopicQueue2(String msg) {
    System.out.println("消费者接收到topic.queue2的消息：【" + msg + "】");
}
```





#### 总结

描述下Direct交换机与Topic交换机的差异？

- Topic交换机接收的消息RoutingKey必须是多个单词，以 `**.**` 分割
- Topic交换机与队列绑定时的bindingKey可以指定通配符
- `#`：代表0个或多个词
- `*`：代表1个词



### 消息转换器

之前说过，Spring会把你发送的消息序列化为字节发送给MQ（前提是对象类型实现序列化接口），接收消息的时候，还会把字节反序列化为Java对象。

![image-20200525170410401](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20200525170410401.png)

只不过，默认情况下Spring采用的序列化方式是JDK序列化。众所周知，JDK序列化存在下列问题：

- 数据体积过大
- 有安全漏洞
- 可读性差

我们来测试一下。



#### 测试默认转换器



我们修改消息发送的代码，发送一个Map对象：

```java
@Test
public void testSendMap() throws InterruptedException {
    // 准备消息
    Map<String,Object> msg = new HashMap<>();
    msg.put("name", "Jack");
    msg.put("age", 21);
    // 发送消息
    rabbitTemplate.convertAndSend("simple.queue","", msg);
}
```



停止consumer服务



发送消息后查看控制台：

![image-20210422232835363](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210422232835363.png)



#### 配置JSON转换器

显然，JDK序列化方式并不合适。我们希望消息体的体积更小、可读性更高，因此可以使用JSON方式来做序列化和反序列化。

在publisher和consumer两个服务中都引入依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

配置消息转换器。

在启动类中添加一个Bean即可：

```java
@Bean
public MessageConverter messageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

但是我个人还是更倾向于自己手动将对象类型转成一个JSON字符串，然后再发送到MQ Server，消费者接收JSON字符串后再手动转成对象。不想用这个。











# 服务异步通信-高级篇



消息队列在使用过程中，面临着很多实际问题需要思考：

![image-20210718155003157](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718155003157.png)





## 消息可靠性

消息从发送，到消费者接收，会经理多个过程：

![image-20210718155059371](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718155059371.png)



其中的每一步都可能导致消息丢失，常见的丢失原因包括：

- 发送时丢失：
  - 生产者发送的消息未送达exchange
  - 消息到达exchange后未到达queue
- MQ宕机，queue将消息丢失
- consumer接收到消息后未消费就宕机



针对这些问题，RabbitMQ分别给出了解决方案：

- 生产者确认机制
- mq持久化
- 消费者确认机制
- 失败重试机制



下面通过案例来演示每一个步骤。案例代码地址：https://github.com/ChanServy/rabbitmq-advanced


### 生产者消息确认

RabbitMQ提供了publisher confirm机制来避免消息发送到MQ过程中丢失。这种机制必须给每个消息指定一个唯一ID。消息发送到MQ以后，会返回一个结果给发送者，表示消息是否处理成功。

返回结果有两种方式：

- publisher-confirm，发送者确认
  - 消息成功投递到交换机，返回ack
  - 消息未投递到交换机，返回nack
- publisher-return，发送者回执
  - 消息投递到交换机了，但是没有路由到队列。返回ACK，及路由失败原因。

![image-20210718160907166](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718160907166.png)



注意：

![image-20210718161707992](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718161707992.png)



#### 修改配置

首先，修改publisher服务中的application.yml文件，添加下面的内容：

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
   
```

说明：

- `publish-confirm-type`：开启publisher-confirm，这里支持两种类型：
  - `simple`：同步等待confirm结果，直到超时
  - `correlated`：异步回调，定义ConfirmCallback，MQ返回结果时会回调这个ConfirmCallback
- `publish-returns`：开启publish-return功能，同样是基于callback机制，不过是定义ReturnCallback
- `template.mandatory`：定义消息路由失败时的策略。true，则调用ReturnCallback；false：则直接丢弃消息



#### 定义Return回调

每个RabbitTemplate只能配置一个ReturnCallback，因此需要在项目加载时配置：

修改publisher服务，添加一个：

```java
package com.chan.mq.producer.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Configuration;

/**
 * 每个RabbitTemplate只能配置一个ReturnCallback，因此需要在项目加载时配置
 * @author CHAN
 * @since 2022/7/7
 */
@Slf4j
@Configuration
public class MyRabbitConfig implements ApplicationContextAware {

    // @Bean
    // public MessageConverter messageConverter() {
    //     return new Jackson2JsonMessageConverter();
    // }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        // 获取RabbitTemplate
        RabbitTemplate rabbitTemplate = applicationContext.getBean(RabbitTemplate.class);
        // 设置ReturnCallback
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            // 判断是否是延迟消息
            Integer receivedDelay = message.getMessageProperties().getReceivedDelay();
            if (receivedDelay != null && receivedDelay > 0) {
                // 是一个延迟消息，忽略这个错误提示
                return;
            }
            // 投递失败，记录日志
            log.info("消息发送失败，应答码{}，原因{}，交换机{}，路由键{},消息{}",
                    replyCode, replyText, exchange, routingKey, message.toString());
            // 如果有业务需要，可以重发消息
            // ...
        });
    }
}

```



#### 定义ConfirmCallback

ConfirmCallback可以在发送消息时指定，因为每个业务处理confirm成功或失败的逻辑不一定相同。

common服务中：

```java
@Data
public class Movie implements Serializable {
    private int id;
    private String name;
    private String author;
}
```



在publisher服务中，定义一个测试方法：

```java
@RequestMapping("/confirm")
public R testSendWithConfirmCallback() {
    // 消息体
    Movie movie = new Movie();
    movie.setId(1);
    movie.setName("复仇者联盟");
    movie.setAuthor("漫威");
    // String movieJson = JSONUtil.toJsonStr(movie);
    // 全局唯一的消息ID，需要封装到CorrelationData中
    CorrelationData correlationData = new CorrelationData(String.valueOf(movie.getId()));
    // 添加callback
    correlationData.getFuture().addCallback(
        result -> {
            // 判断结果
            if (Objects.requireNonNull(result).isAck()) {
                // ACK
                log.debug("消息成功投递到交换机！消息ID: {}", correlationData.getId());
            } else {
                // NACK
                log.error("消息投递到交换机失败！消息ID：{}，原因：{}", correlationData.getId(), result.getReason());
                // 重发消息
            }
        }, 
        exception -> {
            // 记录日志
            log.error("消息发送异常, 消息ID：{}，原因：{}", correlationData.getId(), exception.getMessage());
            // 重发消息...
        }
    );
    // 发送消息
    rabbitTemplate.convertAndSend("simple.queue", movie, correlationData);
    return R.ok();
}
```

消费者：

```java
@Component
@RabbitListener(queues = {"simple.queue"})
public class SimpleQueueListener {

    @RabbitHandler
    public void listenSimpleQueueMessage(Movie movie, Message message, Channel channel) throws IOException {
        System.out.println("spring 消费者接收到Movie类型的消息：【" + movie + "】");
        byte[] body = message.getBody();// 获取消息体
        MessageProperties properties = message.getMessageProperties();// 获取消息头属性信息
        long deliveryTag = properties.getDeliveryTag();// channel内按顺序自增的
        System.out.println("deliveryTag===>" + deliveryTag);
        channel.basicAck(deliveryTag, false);// 消费者手动ACK，false：只手动ACK当前这条消息
        // requeue=false 丢弃；requeue=true 重新发回MQ服务器，重新入队
        channel.basicNack(deliveryTag, false, false);// 消费者手动NACK，false：只手动NACK当前这条消息，false：设置NACK这条消息不重新入队
        channel.basicReject(deliveryTag, false);// 和NACK1个意思，只不过不能设置批量参数
    }
}
```

参数可以写以下类型：

- 1、Message message：原生消息详细信息。头 + 体
- 2、T<发送的消息的类型> ：例如 Book
- 3、Channel channel：当前传输数据的通道。注：如果消费者需要手动 ACK，那么就得传 channel，channel 可以调用相关 API。

后面详细说消费者端消息安全性的问题。



### 消息持久化

生产者确认可以确保消息投递到RabbitMQ的队列中，但是消息发送到RabbitMQ以后，如果突然宕机，也可能导致消息丢失。

要想确保消息在RabbitMQ中安全保存，必须开启消息持久化机制。

- 交换机持久化
- 队列持久化
- 消息持久化



#### 交换机持久化

RabbitMQ中交换机默认是非持久化的，mq重启后就丢失。

SpringAMQP中可以通过代码指定交换机持久化：

```java
@Bean
public DirectExchange simpleExchange(){
    // 三个参数：交换机名称、是否持久化、当没有queue与其绑定时是否自动删除
    return new DirectExchange("simple.direct", true, false);
}
```

事实上，默认情况下，由SpringAMQP声明的交换机都是持久化的。



可以在RabbitMQ控制台看到持久化的交换机都会带上`D`的标示：

![image-20210718164412450](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718164412450.png)



#### 队列持久化

RabbitMQ中队列默认是非持久化的，mq重启后就丢失。

SpringAMQP中可以通过代码指定交换机持久化：

```java
@Bean
public Queue simpleQueue(){
    // 使用QueueBuilder构建队列，durable就是持久化的
    return QueueBuilder.durable("simple.queue").build();
}
```

事实上，默认情况下，由SpringAMQP声明的队列都是持久化的。

可以在RabbitMQ控制台看到持久化的队列都会带上`D`的标示：

![image-20210718164729543](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718164729543.png)



#### 消息持久化

利用SpringAMQP发送消息时，可以设置消息的属性（MessageProperties），指定delivery-mode：

- 1：非持久化
- 2：持久化

用java代码指定：

![image-20210718165100016](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718165100016.png)



默认情况下，SpringAMQP发出的任何消息都是持久化的，不用特意指定。





### 消费者消息确认

RabbitMQ是**阅后即焚**机制，RabbitMQ确认消息被消费者消费后会立刻删除。

而RabbitMQ是通过消费者回执来确认消费者是否成功处理消息的：消费者获取消息后，应该向RabbitMQ发送ACK回执，表明自己已经处理消息。



设想这样的场景：

- 1）RabbitMQ投递消息给消费者
- 2）消费者获取消息后，返回ACK给RabbitMQ
- 3）RabbitMQ删除消息
- 4）消费者宕机，消息尚未处理

这样，消息就丢失了。因此消费者返回ACK的时机非常重要。



而SpringAMQP则允许配置三种确认模式：

- manual：手动ack，需要在业务代码结束后，调用api发送ack。
- auto：自动ack，由spring监测listener代码是否出现异常，没有异常则返回ack；抛出异常则返回nack
- none：关闭ack，MQ假定消费者获取消息后会成功处理，因此消息投递后立即被删除



由此可知：

- none模式下，消息投递是不可靠的，可能丢失
- auto模式类似事务机制，出现异常时返回nack，消息回滚到mq；没有异常，返回ack
- manual：自己根据业务情况，判断什么时候该ack

一般，我们都是使用默认的auto即可。



#### 演示none模式

修改consumer服务的application.yml文件，添加下面内容：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: none # 关闭ack
```

修改consumer服务中的方法，模拟一个消息处理异常：

```java
@RabbitListener(queues = "simple.queue")
public void listenSimpleQueue(String msg) {
    log.info("消费者接收到simple.queue的消息：【{}】", msg);
    // 模拟异常
    System.out.println(1 / 0);
    log.debug("消息处理完成！");
}
```

测试可以发现，当消息处理抛异常时，消息依然被RabbitMQ删除了。



#### 演示auto模式

再次把确认机制修改为auto:

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto # 自动ack
```

在异常位置打断点，再次发送消息，程序卡在断点时，可以发现此时消息状态为unack（未确定状态）：

![image-20210718171705383](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718171705383.png)

抛出异常后，因为Spring会自动返回nack，所以消息恢复至Ready状态，并且没有被RabbitMQ删除：

![image-20210718171759179](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718171759179.png)



### 消费失败重试机制

当消费者出现异常后，消息会不断requeue（重入队）到队列，再重新发送给消费者，然后再次异常，再次requeue，无限循环，导致mq的消息处理飙升，带来不必要的压力：

![image-20210718172746378](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718172746378.png)

怎么办呢？



#### 本地重试

我们可以利用Spring的retry机制，在消费者收到消息后，运行过程中消费者出现异常时利用本地重试，而不是无限制的requeue到mq队列。

修改consumer服务的application.yml文件，添加内容：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true # 开启消费者失败重试
          initial-interval: 1000 # 初始的失败等待时长为1秒
          multiplier: 1 # 失败的等待时长倍数，下次等待时长 = multiplier * last-interval
          max-attempts: 3 # 最大重试次数
          stateless: true # true无状态；false有状态。如果业务中包含事务，这里改为false
```



重启consumer服务，重复之前的测试。可以发现：

- 在重试3次后，SpringAMQP会抛出异常AmqpRejectAndDontRequeueException，说明本地重试触发了
- 查看RabbitMQ控制台，发现消息被删除了，说明最后SpringAMQP返回的是ack，mq删除消息了



结论：

- 开启本地重试时，消息处理过程中抛出异常，不会requeue到队列，而是在消费者本地重试
- 重试达到最大次数后，Spring会返回ack，消息会被丢弃



#### 失败策略

在之前的测试中，达到最大重试次数后，消息会被丢弃，这是由Spring内部机制决定的。

在开启重试模式后，重试次数耗尽，如果消息依然失败，则需要有MessageRecovery接口来处理，它包含三种不同的实现：

- RejectAndDontRequeueRecoverer：重试耗尽后，直接reject，丢弃消息。默认就是这种方式

- ImmediateRequeueMessageRecoverer：重试耗尽后，返回nack，消息重新入队

- RepublishMessageRecoverer：重试耗尽后，将失败消息投递到指定的交换机



比较优雅的一种处理方案是RepublishMessageRecoverer，失败后将消息投递到一个指定的，专门存放异常消息的队列，后续由人工集中处理。



1）在consumer服务中定义处理失败消息的交换机和队列

```java
@Bean
public DirectExchange errorMessageExchange(){
    return new DirectExchange("error.direct");
}
@Bean
public Queue errorQueue(){
    return new Queue("error.queue", true);
}
@Bean
public Binding errorBinding(Queue errorQueue, DirectExchange errorMessageExchange){
    return BindingBuilder.bind(errorQueue).to(errorMessageExchange).with("error");
}
```



2）定义一个RepublishMessageRecoverer，关联队列和交换机

```java
@Bean
public MessageRecoverer republishMessageRecoverer(RabbitTemplate rabbitTemplate){
    return new RepublishMessageRecoverer(rabbitTemplate, "error.direct", "error");
}
```



完整代码：

```java
package com.chan.mq.consumer.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.retry.MessageRecoverer;
import org.springframework.amqp.rabbit.retry.RepublishMessageRecoverer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 消费者自动ACK的情况下，当消费者有异常，Spring会感知到RabbitListener中的异常，不会ACK并且会将消息requeue回原来的队列中。
 * 但是requeue回队列中就又会立刻被消费者收到并且异常，这样就会一直requeue，从而降低MQ的吞吐量。因此我们配置了Spring本地重试，
 * 这样本地重试一定次数（我们可以配置）后，SpringAMQP会抛出AmqpRejectAndDontRequeueException（说明本地重试触发了消息不会requeue了）
 * 并且SpringAMQP会返回ACK给RabbitMQ服务器将消息删除。
 *
 * 结论：
 * 开启本地重试时，消息处理过程中抛出异常，不会requeue到队列，而是在消费者本地重试
 * 重试达到最大次数后，Spring会返回ack，相当于Spring默认本地重试一定次数后消息一定会被成功消费，消息会被丢弃
 *
 * 这个丢弃策略是默认的，但是这样就会造成消息丢失，并不好，因此我们需要有一个MessageRecovery接口来处理，这个接口包含3种实现：
 * RejectAndDontRequeueRecoverer：重试耗尽后，直接reject，丢弃消息。默认就是这种方式
 * ImmediateRequeueMessageRecoverer：重试耗尽后，返回nack，消息重新入队
 * RepublishMessageRecoverer：重试耗尽后，将失败消息投递到指定的交换机
 *
 * 我们选择RepublishMessageRecoverer，处理消息失败后将消息投递到一个指定的，专门存放异常消息的队列，后续由人工集中处理。
 * 配置就是本类。
 *
 * 当消费者出现异常了，异常消息的处理方案可以使用这种，也可以使用死信交换机那种。
 *
 * @author CHAN
 * @since 2022/7/8
 */
@Configuration
public class RepublishMessageRecovererConfig {
    /**
     * 专门存放处理失败的消息的交换机
     * @return
     */
    @Bean
    public DirectExchange errorMessageExchange(){
        return new DirectExchange("error.direct");
    }

    /**
     * 存放异常消息的队列
     * @return
     */
    @Bean
    public Queue errorQueue(){
        return new Queue("error.queue", true);
    }

    /**
     * 绑定
     * @return
     */
    @Bean
    public Binding errorBinding(){
        return BindingBuilder.bind(errorQueue()).to(errorMessageExchange()).with("error");
    }
    /**
     * 定义一个RepublishMessageRecoverer，关联队列和交换机
     */
    @Bean
    public MessageRecoverer republishMessageRecoverer(RabbitTemplate rabbitTemplate){
        return new RepublishMessageRecoverer(rabbitTemplate, "error.direct", "error");
    }
}
```





### 总结

如何确保RabbitMQ消息的可靠性？

- 开启生产者确认机制，确保生产者的消息能到达队列
- 开启持久化功能，确保消息未消费前在队列中不会丢失
- 开启消费者确认机制为auto，由spring确认消息处理成功后完成ack
- 开启消费者失败重试机制，并设置MessageRecoverer，多次重试失败后将消息投递到异常交换机，交由人工处理





## 死信交换机



### 初识死信交换机



#### 什么是死信交换机

什么是死信？

当一个队列中的消息满足下列情况之一时，可以成为死信（dead letter）：

- 消息被消费者使用reject拒绝（丢弃）了或使用nack声明消费失败了，并且消息的requeue参数设置为false
  - 情况一：比如收到消息后消费者出现异常，本地重试相应次数后的默认策略（上面介绍过），消息reject情况。
  - 情况二：比如设置了手动ACK机制。消费者端手动NACK了，并且requeue参数设置为了false。消息requeue=false情况。
- 消息是一个过期消息，超时无人消费
- 要投递的队列消息满了，无法投递



如果这个包含死信的队列配置了`dead-letter-exchange`属性，指定了一个交换机，那么队列中的死信就会投递到这个交换机中，而这个交换机称为**死信交换机**（Dead Letter Exchange，检查DLX）。



如图，一个消息被消费者拒绝了，变成了死信：

![image-20210718174328383](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718174328383.png)

因为simple.queue绑定了死信交换机 dl.direct，因此死信会投递给这个交换机：

![image-20210718174416160](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718174416160.png)

如果这个死信交换机也绑定了一个队列，则消息最终会进入这个存放死信的队列：

![image-20210718174506856](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718174506856.png)



另外，队列将死信投递给死信交换机时，必须知道两个信息：

- 死信交换机名称
- 死信交换机与死信队列绑定的RoutingKey

这样才能确保投递的消息能到达死信交换机，并且正确的路由到死信队列。

![image-20210821073801398](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210821073801398.png)





#### 利用死信交换机接收死信（拓展）

在失败重试策略中，默认的RejectAndDontRequeueRecoverer会在本地重试次数耗尽后，发送reject给RabbitMQ，消息变成死信，被丢弃。



我们可以给simple.queue添加一个死信交换机，给死信交换机绑定一个队列。这样消息变成死信后也不会丢弃，而是最终投递到死信交换机，路由到与死信交换机绑定的队列。



![image-20210718174506856](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718174506856.png)



我们在consumer服务中，定义一组死信交换机、死信队列：

```java
package com.chan.mq.consumer.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author CHAN 注：与图中不一致。图只是思路，这个类是我自己配置的。
 * @since 2022/7/8
 */
@Configuration
public class DeadLetterConfig {
    // 声明普通的queue队列，并且为其指定死信交换机：dl.direct
    // 这样这个普通的队列中如果有死信，那么死信就会进到这个死信交换机
    @Bean
    public Queue normalQueue() {
        return QueueBuilder.durable("normal.queue")// 指定队列名称，并持久化
                .deadLetterExchange("dl.direct")// 指定死信交换机
                .deadLetterRoutingKey("dl")// 指定死信交换机和死信队列绑定的key
                .build();
    }
    // 声明死信交换机 dl.direct
    @Bean
    public DirectExchange dlExchange(){
        return new DirectExchange("dl.direct", true, false);
    }
    // 声明存储死信的队列 dl.queue
    @Bean
    public Queue dlQueue(){
        return new Queue("dl.queue", true);
    }
    // 将死信队列 与 死信交换机绑定
    @Bean
    public Binding dlBinding(){
        return BindingBuilder.bind(dlQueue()).to(dlExchange()).with("dl");
    }
}

```









#### 总结

什么样的消息会成为死信？

- 消息被消费者reject或者返回nack
- 消息超时未消费
- 队列满了

如何给队列指定死信交换机？

- 给队列设置dead-letter-exchange属性，指定一个交换机
- 给队列设置dead-letter-routing-key属性，设置死信交换机与死信队列的RoutingKey

死信交换机的使用场景是什么？

- 如果队列绑定了死信交换机，死信会投递到死信交换机；
- 可以利用死信交换机收集所有消费者处理失败的消息（死信），交由人工处理，进一步提高消息队列的可靠性。



### TTL

一个队列中的消息如果超时未消费，则会变为死信，超时分为两种情况：

- 消息所在的队列设置了超时时间
- 消息本身设置了超时时间

![image-20210718182643311](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718182643311.png)



![2022-06-24_22-05.png](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/2022-06-24_22-05.png)

#### 接收超时死信的死信交换机

在consumer服务中，定义一个新的消费者，并且声明死信交换机、死信队列，这个消费者消费死信队列中的死信，也就是正常队列超时或者正常队列中的消息超时从而进到死信交换机（前提是这个正常队列创建时指定了死信交换机），进而进到死信队列中的消息，业务中可看做延迟消息。

注解方式声明死信交换机和死信队列：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "dl.ttl.queue", durable = "true"),
    exchange = @Exchange(name = "dl.ttl.direct"),
    key = "dl.ttl"
))
public void listenDlQueue(String msg){
    log.info("接收到 dl.ttl.queue的延迟消息：{}", msg);
}
```



#### 声明一个队列，并且指定TTL

要给队列设置超时时间，需要在声明队列时配置x-message-ttl属性：

```java
@Bean
public Queue ttlQueue(){
    return QueueBuilder.durable("ttl.queue") // 指定普通队列名称，并持久化
        .ttl(10000) // 设置队列的超时时间，10秒
        .deadLetterExchange("dl.ttl.direct") // 指定死信交换机
        .deadLetterRoutingKey("dl.ttl")// 指定死信交换机和死信队列绑定的key
        .build();
}
```

注意，这个队列设定了死信交换机为`dl.ttl.direct`



声明交换机，将ttl.queue与交换机绑定：

```java
@Bean
public DirectExchange ttlExchange(){
    return new DirectExchange("ttl.direct");
}
@Bean
public Binding ttlBinding(){
    return BindingBuilder.bind(ttlQueue()).to(ttlExchange()).with("ttl");
}
```



或者使用@Bean的方式声明死信交换机和死信队列也可。

下面是使用@Bean的方式声明死信交换机和死信队列并且使用@Bean的方式创建可超时的普通队列的整体代码：

```java
package com.chan.mq.consumer.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * TTL延时消息也是基于死信交换机和死信队列实现
 *
 * @author CHAN
 * @since 2022/7/8
 */
@Configuration
public class TTLMessageWithDLConfig {
    // 声明一个队列，并且指定TTL，这个队列设定了死信交换机为dl.ttl.direct
    @Bean
    public Queue ttlQueue() {
        return QueueBuilder.durable("ttl.queue")// 指定普通队列名称，并持久化
                .ttl(10000)// 设置队列的超时时间，10秒，普通队列超时，其中的消息就会变为死信进入死信交换机，我们只需要监听死信队列就可以实现延迟消息的效果
                .deadLetterExchange("dl.ttl.direct")// 指定死信交换机
                .deadLetterRoutingKey("dl.ttl")// 指定死信交换机和死信队列绑定的key
                .build();
    }
    // 声明交换机，将ttl.queue与交换机绑定
    @Bean
    public DirectExchange ttlExchange() {
        return new DirectExchange("ttl.direct");
    }
    @Bean
    public Binding ttlBinding() {
        return BindingBuilder.bind(ttlQueue()).to(ttlExchange()).with("ttl");
    }
    // 声明死信交换机
    @Bean
    public DirectExchange dlTTLExchange() {
        return new DirectExchange("dl.ttl.direct");
    }
    // 声明死信队列
    @Bean
    public Queue dlTTLQueue() {
        return new Queue("dl.ttl.queue", true);
    }
    // 绑定死信队列到死信交换机
    @Bean
    public Binding dlTTLBinding() {
        return BindingBuilder.bind(dlTTLQueue()).to(dlTTLExchange()).with("dl.ttl");
    }
}

```

```java
@Component
@Slf4j
public class DlTtlQueueListener {
    @RabbitListener(queues = {"dl.ttl.queue"})
    public void listenDlQueue(String msg){
        log.info("接收到 dl.ttl.queue的延迟消息：{}", msg);
    }
}
```





发送消息，但是不要指定TTL：

```java
@RequestMapping("/ttlqueue")
public R testSendToTTLQueue() {
    // 创建消息
    String message = "hello, ttl queue";
    // 消息ID，需要封装到CorrelationData中
    CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
    // 发送消息
    rabbitTemplate.convertAndSend("ttl.direct", "ttl", message, correlationData);
    // 记录日志
    log.debug("发送消息成功");
    return R.ok();
}
```

发送消息的日志：

![image-20210718191657478](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718191657478.png)



查看下接收消息的日志：

![image-20210718191738706](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718191738706.png)



因为队列的TTL值是10000ms，也就是10秒。可以看到消息发送与接收之间的时差刚好是10秒。



#### 发送消息时，设定TTL

```java
@Configuration
public class DeadLetterConfig {
    // 声明普通的queue队列，并且为其指定死信交换机：dl.direct
    // 这样这个普通的队列中如果有死信，那么死信就会进到这个死信交换机
    @Bean
    public Queue normalQueue() {
        return QueueBuilder.durable("normal.queue")// 指定队列名称，并持久化
                .deadLetterExchange("dl.direct")// 指定死信交换机
                .deadLetterRoutingKey("dl")// 指定死信交换机和死信队列绑定的key
                .build();
    }
    // 声明死信交换机 dl.direct
    @Bean
    public DirectExchange dlExchange(){
        return new DirectExchange("dl.direct", true, false);
    }
    // 声明存储死信的队列 dl.queue
    @Bean
    public Queue dlQueue(){
        return new Queue("dl.queue", true);
    }
    // 将死信队列 与 死信交换机绑定
    @Bean
    public Binding dlBinding(){
        return BindingBuilder.bind(dlQueue()).to(dlExchange()).with("dl");
    }
}
```

```java
@Component
@Slf4j
public class DlQueueListener {
    @RabbitListener(queues = {"dl.queue"})
    public void listenDlQueue(String msg){
        log.info("接收到 dl.queue的延迟消息：{}", msg);
    }
}
```



在发送消息时，也可以指定TTL：

```java
/**
 * 由于我们需要设置消息的TTL超时时间，因此需要使用MessageBuilder的方式发送消息。当使用MessageBuilder发送消息到MQ时，会有一个问题：
 * 就是当我们项目中同时配置了Jackson2JsonMessageConverter(将对象类型的消息序列化成JSON)，消息序列化时会出错导致异常。
 * 取消这个序列化配置，就不会出错了，但是取消序列化配置之后，发送对象类型的消息（比如消息体是一个Book类型）时又会报错。
 * 显示SimpleMessageConverter只能转换String、字节数组等基本类型，因此我们可以让这个对象实现Serializable；
 * 但是不配置JSON序列化，而使用Java的Serializable序列化的话，将对象消息发送到MQ服务器时可读性太差。
 * 因此，最好的方式是我们自己将对象序列化成JSON字符串之后，再将对象的JSON字符串发送到MQ，消费者收到JSON之后再将JSON序列化成对象包装类。
 * 简单来说就是如果我们想发送一个我们自己的对象类型到MQ，我们自己完成JSON序列化和反序列化，不用通过配置让MQ帮我们完成。
 * @return R
 */
@RequestMapping("/ttlmsg")
public R testSendTTLMsgToQueue() {
    // 创建消息
    Message message = MessageBuilder
        .withBody("hello, ttl message".getBytes(StandardCharsets.UTF_8))
        .setDeliveryMode(MessageDeliveryMode.PERSISTENT)
        .setExpiration("5000")
        .build();
    // 消息ID，需要封装到CorrelationData中
    CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
    // 发送消息
    rabbitTemplate.convertAndSend("normal.queue", message, correlationData);
    log.debug("发送消息成功");
    return R.ok();
}
```

**注：消息生产者发送的消息类型和消息消费者接收的消息类型要一致！**



这次，发送与接收的延迟只有5秒。说明当队列、消息都设置了TTL时，任意一个到期就会成为死信。



#### 总结

消息超时的两种方式是？

- 给队列设置ttl属性，进入队列后超过ttl时间的消息变为死信
- 给消息设置ttl属性，队列接收到消息超过ttl时间后变为死信

如何实现发送一个消息20秒后消费者才收到消息？

- 给消息的目标队列指定死信交换机
- 将消费者监听的队列绑定到死信交换机
- 发送消息时给消息设置超时时间为20秒



### 延迟队列

利用TTL结合死信交换机，我们实现了消息发出后，消费者延迟收到消息的效果。这种消息模式就称为延迟队列（Delay Queue）模式。

延迟队列的使用场景包括：

- 延迟发送短信
- 用户下单，如果用户在15 分钟内未支付，则自动取消
- 预约工作会议，20分钟后自动通知所有参会人员



因为延迟队列的需求非常多，所以RabbitMQ的官方也推出了一个插件，原生支持延迟队列效果。

这个插件就是DelayExchange插件。参考RabbitMQ的插件列表页面：https://www.rabbitmq.com/community-plugins.html

![image-20210718192529342](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718192529342.png)



使用方式可以参考官网地址：https://blog.rabbitmq.com/posts/2015/04/scheduling-messages-with-rabbitmq



#### 安装DelayExchange插件

https://chanservy.vercel.app/posts/20220622/rabbitmq-install.html

#### DelayExchange原理

DelayExchange需要将一个交换机声明为delayed类型。当我们发送消息到delayExchange时，流程如下：

- 接收消息
- 判断消息是否具备x-delay属性
- 如果有x-delay属性，说明是延迟消息，持久化到硬盘，读取x-delay值，作为延迟时间
- 返回routing not found结果给消息发送者
- x-delay时间到期后，重新投递消息到指定队列



#### 使用DelayExchange

插件的使用也非常简单：声明一个交换机，交换机的类型可以是任意类型，只需要设定delayed属性为true即可，然后声明队列与其绑定即可。

##### 1）声明DelayExchange交换机

基于注解方式（推荐）：

![image-20210718193747649](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718193747649.png)

```java
/**
 * 创建并监听延迟队列
 *
 * 延时队列可以使用这种，也可以使用死信交换机那种。
 *
 * @author CHAN
 * @since 2022/7/8
 */
@Component
@Slf4j
public class DelayQueueListener {
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "delay.queue", durable = "true"),
            exchange = @Exchange(name = "delay.direct", delayed = "true"),
            key = "delay"
    ))
    public void listenDelayExchange(String msg) {
        log.info("消费者接收到了delay.queue的延迟消息:{}", msg);
    }
}
```



也可以基于@Bean的方式：

![image-20210718193831076](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718193831076.png)



##### 2）发送消息

发送消息时，一定要携带x-delay属性，指定延迟的时间：

![image-20210718193917009](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718193917009.png)

```java
@RequestMapping("/delay")
public R testDelayQueue() {
    // 1.准备消息
    Book book = new Book();
    book.setName("蛤蟆先生去看心理医生");
    book.setAuthor("罗伯特");
    String bookJson = JSONUtil.toJsonStr(book);
    Message message = MessageBuilder
        .withBody(bookJson.getBytes(StandardCharsets.UTF_8))
        .setDeliveryMode(MessageDeliveryMode.PERSISTENT)
        .setHeader("x-delay", 5000)
        .build();
    // 2.准备CorrelationData
    CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
    // 3.发送消息
    rabbitTemplate.convertAndSend("delay.direct", "delay", message, correlationData);

    log.info("发送消息成功");
    return R.ok();
}
```



#### 总结

延迟队列插件的使用步骤包括哪些？

- 声明一个交换机，添加delayed属性为true
- 发送消息时，添加x-delay头，值为超时时间



## 惰性队列

### 消息堆积问题

当生产者发送消息的速度超过了消费者处理消息的速度，就会导致队列中的消息堆积，直到队列存储消息达到上限。之后发送的消息就会成为死信，可能会被丢弃，这就是消息堆积问题。



![image-20210718194040498](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718194040498.png)





解决消息堆积有两种思路：

- 增加更多消费者，提高消费速度。也就是我们之前说的work queue模式
- 扩大队列容积，提高堆积上限



要提升队列容积，把消息保存在内存中显然是不行的。



### 惰性队列

从RabbitMQ的3.6.0版本开始，就增加了Lazy Queues的概念，也就是惰性队列。惰性队列的特征如下：

- 接收到消息后直接存入磁盘而非内存
- 消费者要消费消息时才会从磁盘中读取并加载到内存
- 支持数百万条的消息存储



#### 基于命令行设置lazy-queue

而要设置一个队列为惰性队列，只需要在声明队列时，指定x-queue-mode属性为lazy即可。可以通过命令行将一个运行中的队列修改为惰性队列：

```sh
rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues  
```

命令解读：

- `rabbitmqctl` ：RabbitMQ的命令行工具
- `set_policy` ：添加一个策略
- `Lazy` ：策略名称，可以自定义
- `"^lazy-queue$"` ：用正则表达式匹配队列的名字
- `'{"queue-mode":"lazy"}'` ：设置队列模式为lazy模式
- `--apply-to queues  `：策略的作用对象，是所有的队列



#### 基于@Bean声明lazy-queue

![image-20210718194522223](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718194522223.png)

#### 基于@RabbitListener声明LazyQueue

![image-20210718194539054](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718194539054.png)





### 总结

消息堆积问题的解决方案？

- 队列上绑定多个消费者，提高消费速度
- 使用惰性队列，可以再mq中保存更多消息

惰性队列的优点有哪些？

- 基于磁盘存储，消息上限高
- 没有间歇性的page-out，性能比较稳定

惰性队列的缺点有哪些？

- 基于磁盘存储，消息时效性会降低
- 性能受限于磁盘的IO





## MQ集群



### 集群分类

RabbitMQ的是基于Erlang语言编写，而Erlang又是一个面向并发的语言，天然支持集群模式。RabbitMQ的集群有两种模式：

•**普通集群**：是一种分布式集群，将队列分散到集群的各个节点，从而提高整个集群的并发能力。

•**镜像集群**：是一种主从集群，普通集群的基础上，添加了主从备份功能，提高集群的数据可用性。



镜像集群虽然支持主从，但主从同步并不是强一致的，某些情况下可能有数据丢失的风险。因此在RabbitMQ的3.8版本以后，推出了新的功能：**仲裁队列**来代替镜像集群，底层采用Raft协议确保主从的数据一致性。



### 普通集群



#### 集群结构和特征

普通集群，或者叫标准集群（classic cluster），具备下列特征：

- 会在集群的各个节点间共享部分数据，包括：交换机、队列元信息。不包含队列中的消息。
- 当访问集群某节点时，如果队列不在该节点，会从数据所在节点传递到当前节点并返回
- 队列所在节点宕机，队列中的消息就会丢失

结构如图：

![image-20210718220843323](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718220843323.png)



#### 部署

https://chanservy.vercel.app/posts/20220622/rabbitmq-install.html



### 镜像集群



#### 集群结构和特征

镜像集群：本质是主从模式，具备下面的特征：

- 交换机、队列、队列中的消息会在各个mq的镜像节点之间同步备份。
- 创建队列的节点被称为该队列的**主节点，**备份到的其它节点叫做该队列的**镜像**节点。
- 一个队列的主节点可能是另一个队列的镜像节点
- 所有操作都是主节点完成，然后同步给镜像节点
- 主宕机后，镜像节点会替代成新的主

结构如图：

![image-20210718221039542](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/image-20210718221039542.png)





#### 部署

https://chanservy.vercel.app/posts/20220622/rabbitmq-install.html



### 仲裁队列



#### 集群特征

仲裁队列：仲裁队列是3.8版本以后才有的新功能，用来替代镜像队列，具备下列特征：

- 与镜像队列一样，都是主从模式，支持主从数据同步
- 使用非常简单，没有复杂的配置
- 主从同步基于Raft协议，强一致



#### 部署

https://chanservy.vercel.app/posts/20220622/rabbitmq-install.html





#### Java代码创建仲裁队列

```java
@Bean
public Queue quorumQueue() {
    return QueueBuilder
        .durable("quorum.queue") // 持久化
        .quorum() // 仲裁队列
        .build();
}
```



#### SpringAMQP连接MQ集群

注意，这里用address来代替host、port方式

```yml
spring:
  rabbitmq:
    addresses: 192.168.150.105:8071, 192.168.150.105:8072, 192.168.150.105:8073
    username: itcast
    password: 123321
    virtual-host: /
```









