---
title: RabbitMQ - 延迟队列
category: 延迟队列
tags:
- 延迟队列
- RabbitMQ
---


延迟队列在我们的日常生活中很常见，比如超时取消订单功能，优惠卷过期，消息延时推送等功能。RabbitMQ是一个广泛的消息队列中间件，它支持延迟队列的功能，我们可以通过设置消息的TTL (过期时间)  结合死信队列实现，或通过RabbitMQ的插件实现延迟队列。



## 1.什么是延迟队列：

**延迟队列**是一个特殊类型的消息队列，其核心特点是任务或消息在被发送者送到队列后，并不会马上被消费，而是等待预设的时间到了后才被消费者消费。在分布式系统中，延迟队列是一个常见的工具，他允许程序能够按照预定的时间处理任务（定时任务）。



## 2.基于RabbitMQ实现延迟队列原理

### 2.1.RabbitMQ中的TTL

首先我们要知道RabbitMQ中的TTL，他是RabbitMQ中的一个消息或者队列的一个属性，表明一条消息或者这个队列中的所有消息的最大存活时间。如果一条消息设置了TTL属性或者进入了设置TTL属性的队列，那这条消息如果**在TTL设置的时间内没有被消费**，就会变为**死信**。(消息TTL和设置了TTL队列如果同时存在，以最小的为准)。

#### 2.1.1.队列设置TTL

在设置队列的时候设置队列的“x-message-ttl”属性

```java
@Bean("QueueA")
public Queue queueA(){
    Map<String, Object> map = new HashMap<>(3);
    //设置死信交换机
    map.put("x-dead-letter-exchange", Exchange_Dead);
    //设置死信路由键
    map.put("x-dead-letter-routing-key", "YD");
    //设置TTL 单位ms
    map.put("x-message-ttl", 10000);
    return QueueBuilder.durable(Queue_A).withArguments(map).build();
}
```

#### 2.1.2.消息设置TTL

对每条消息设置TTL

```java
        Message msg = MessageBuilder.withBody(message.getBytes(StandardCharsets.UTF_8))
                .setExpiration(String.valueOf(ttl)) //设置TTL 单位ms
                .build();
        rabbitTemplate.convertAndSend("X", "XC", msg);
```

#### 2.1.3.两者的区别

通过队列设的TTL属性，一旦消息过期就会被丢弃(如果配置了死信队列就会被丢到**死信队列**中)，而通过消息设置TTL，消息到期也不一定会马上被丢弃，主要是队列的先进先出特性，只有过期的消息到了队列的队首，才会被丢弃或进入死信队列，这样如果队列前面有很多未到期的消息，但是后面后来的消息已经到期了但是前面的消息还有没有到期，造成消息积压，这样即使已经过期的消息也许还能存活较长时间。



## 3.RabbitMQ实现延迟队列

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250311232201.png)

前面我们简绍了RabbitMQ中的TTL属性，基于这个属性我们可以结合死信队列实现延迟队列，当生产者发送消息到一个没有消费者的队列，这个消息TTL到期后变成死信消息进入死信队列然后被消费者消费，这样是不是就是实现了消息不会马上被消费，而是等TTL（预设时间）到期后才被消费。

### 3.1.队列TTL

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312161238.png)

创建两个队列QA和QB,两个队列分别设置TTL10S和30S，然后创建一个交换机X和死信交换机Y，由于QA和QD没有指定消费者所有消息最终会变为死信进入死信队列，然后被延迟消费。

#### 3.1.1.配置和代码

**项目结构：**

rabbitmq
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com.luyseon.rabbitmq
│   │   │       ├── config
│   │   │       │   └── rabbitmqConfig.java
│   │   │       ├── consumer
│   │   │       │   └── DelayQueueConsumer.java
│   │   │       ├── controller
│   │   │       │   └── publishController.java
│   │   │       └── RabbitmqApplication.java
│   │   └── resources
│   │       └── application.yml
│   └── test
│       └── (测试代码目录)
└── (其他项目文件)

**通过Bean统一声名交换机和队列：**

```java
@Configuration
public class rabbitmqConfig {

    //交换机
    public static final String Exchange_X = "X";
    //死信交换机
    public static final String Exchange_Dead = "Y";
    //队列
    public static final String Queue_A = "QA";
    public static final String Queue_B = "QB";
    //死信队列
    public static final String Queue_Dead = "QD";

    //声明交换机X
    @Bean("ExchangeX")
    public DirectExchange exchangeX(){
        return new DirectExchange(Exchange_X);
    }

    //声明死信交换机Y
    @Bean("ExchangeY")
    public DirectExchange exchangeDead(){
        return new DirectExchange(Exchange_Dead);
    }

    //声明队列A
    @Bean("QueueA")
    public Queue queueA(){
        Map<String, Object> map = new HashMap<>(3);
        //设置死信交换机
        map.put("x-dead-letter-exchange", Exchange_Dead);
        //设置死信路由键
        map.put("x-dead-letter-routing-key", "YD");
        //设置TTL 单位ms
        map.put("x-message-ttl", 10000);
        return QueueBuilder.durable(Queue_A).withArguments(map).build();
    }

    //声明队列B
    @Bean("QueueB")
    public Queue queueB(){
        Map<String, Object> map = new HashMap<>(3);
        //设置死信交换机
        map.put("x-dead-letter-exchange", Exchange_Dead);
        //设置死信路由键
        map.put("x-dead-letter-routing-key", "YD");
        //设置TTL 单位ms
        map.put("x-message-ttl", 30000);
        return QueueBuilder.durable(Queue_B).withArguments(map).build();
    }

    //声明死信队列
    @Bean("QueueDead")
    public Queue queueDead(){
        return QueueBuilder.durable(Queue_Dead).build();
    }

    //绑定队列A到交换机X的路由键为XA
    @Bean
    public Binding bindingXA(@Qualifier("QueueA") Queue queueA,
                             @Qualifier("ExchangeX") DirectExchange exchangeX){
       return BindingBuilder.bind(queueA).to(exchangeX).with("XA");
    }

    //绑定队列B到交换机X的路由键为XB
    @Bean
    public Binding bindingXB(@Qualifier("QueueB") Queue queueB,
                             @Qualifier("ExchangeX") DirectExchange exchangeX){
       return BindingBuilder.bind(queueB).to(exchangeX).with("XB");
    }

    //绑定死信队列到死信交换机Y的路由键为YD
    @Bean
    public Binding bindingDead(@Qualifier("QueueDead") Queue queueDead,
                               @Qualifier("ExchangeY") DirectExchange exchangeDead){
       return BindingBuilder.bind(queueDead).to(exchangeDead).with("YD");
    }


}
```

**消息发送者：**

```java
@Slf4j
@RestController
@RequestMapping("/ttl")
public class publishController {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping("/sendMsg/{message}")
    public void sendMsg(@PathVariable String message) {
        log.info("当前时间：{},发送一条信息给两个TTL队列:{}", new Date(), message);
        rabbitTemplate.convertAndSend("X", "XB","ttl为30s的队列：" + message);
        rabbitTemplate.convertAndSend("X", "XA","ttl为10s的队列：" + message);
    }
}
```

**监听消息消费者：**

```java
@Slf4j
@Component
public class DelayQueueConsumer {

    // 监听消息
    @RabbitListener(queues = "QD")
    public void receiveDelayQueue(Message message) {
        String msg = new String(message.getBody());
        log.info("当前时间：{}，收到延迟队列的消息：{}", new Date().toString(), msg);
    }
}
```

通过浏览器访问：http://localhost:8080/ttl/sendMsg/{hhhh} 可以看到日志打印：

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312195652.png)

这样成功实现了延迟队列，但是如果这样使用，每需要一个新的时间需求的延迟，就需要我们新增一个队列 ，这显然不行。

### 3.2.消息TTL

我们新增一个队列QC,该队列不设置TTL时间。这个QC队列是通用的，我们通过生产者发送消息时设置TTL。

```java
@Configuration
public class rabbitmqConfig {

	//前面一样..

    //声名队列C（队列不设置TTL）
    @Bean("QueueC")
    public Queue queueC(){
        HashMap<String, Object> map = new HashMap<>(3);
        map.put("x-dead-letter-exchange", Exchange_Dead);
        map.put("x-dead-letter-routing-key", "YD");

        return QueueBuilder.durable(Queue_C).withArguments(map).build();
    }

    //绑定队列C到交换机X的路由键为XC
    @Bean
    public Binding bindingC(@Qualifier("QueueC") Queue queueC,
                            @Qualifier("ExchangeX") DirectExchange exchangeX){
        return BindingBuilder.bind(queueC).to(exchangeX).with("XC");
    }

}
```

消息发送者：

```java
@GetMapping("/sendMsg/{message}/{ttl}")
public void sendMsg(@PathVariable String message, @PathVariable int ttl) {
    log.info("当前时间：{}，ttl为{}ms的队列：{}", new Date(), ttl, message);
    Message msg = MessageBuilder.withBody(message.getBytes(StandardCharsets.UTF_8))
            .setExpiration(String.valueOf(ttl))  //设置消息TTL
            .build();
    rabbitTemplate.convertAndSend("X", "XC",msg);
}
```

我们发送两个请求

http://localhost:8080/ttl/sendMsg/你好 1/20000

http://localhost:8080/ttl/sendMsg/你好 2/10000

按理来说应该第二个请求先到达死信队列，但是结果却是：

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312204454.png)

造成这样的情况就是前面所说的消息挤压的问题，第一个消息的延时时间很长，而第二个消息的延时时间很短，由于队列的先进先出特性，消息只有到达队列顶端才会被消费。

### 3.3.使用RabbitMQ插件优化

针对这个问题，我们可以使用RabbitMQ官方提供的一个插件解决。在官网上 https://www.rabbitmq.com/community-plugins.html, 下载 rabbitmq_delayed_message_exchange 插件，然后解压放置到 RabbitMQ 的插件目录。 进入 RabbitMQ 的安装目录下的 plgins 目录，执行下面命令让该插件生效，然后重启 RabbitMQ，具体教程这里不做过多讲解。

安装好后我们在RabbitMQ的新增交换机页面的Type选可以看见这个**x-delayed-message**:

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312205920.png)

通过这个插件我们消息延迟的位置将不是在队列，而是在交换机，TTL到达后会进入队列，然后被消费。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312210337.png)



#### 3.3.1.配置和代码

我们重新配置一个交换机 **delayed.exchange** 和队列 **delayed.queue**。

**config代码：**

```java
@Configuration
public class delayedConfig {

    public static final String Delayed_Exchange = "delayed.exchange";
    public static final String Delayed_Queue = "delayed.queue";
    
    public static final String Delayed_Routing_Key = "delayed.routingKey";
    
    @Bean
    public CustomExchange delayedExchange() {
        HashMap<String, Object> arguments = new HashMap<>();

        arguments.put("x-delayed-type", "direct");// 声明交换机的类型，以及延迟消息的类型（direct直接型）
        return new CustomExchange(Delayed_Exchange, "x-delayed-message", true, false, arguments);
    }

    @Bean
    public Queue delayedQueue() {
        return new Queue(Delayed_Queue);
    }

    @Bean
    public Binding bindingDelayed(@Qualifier("delayedQueue") Queue delayedQueue,
                                  @Qualifier("delayedExchange") CustomExchange delayedExchange) {
        return BindingBuilder.bind(delayedQueue).to(delayedExchange).with(Delayed_Routing_Key).noargs();
    }

}
```

**发送者：**

```java
@GetMapping("/sendDelayMsg/{message}/{delayTime}")
public void sendMsgDelay(@PathVariable String message, @PathVariable Integer delayTime) {
    log.info("当前时间：{}，发送一条信息给延迟队列：{}", new Date(), message);
    rabbitTemplate.convertAndSend(delayedConfig.Delayed_Exchange, delayedConfig.Delayed_Routing_Key, message, msg -> {
        msg.getMessageProperties().setDelayLong(Long.valueOf(delayTime));
        return msg;
    });
}
```

**消息监听：**

```java
// delayed监听消息
@RabbitListener(queues = "delayed.queue")
public void receiveDelayedQueue(Message message) {
    String msg = new String(message.getBody());
    log.info("当前时间：{}，收到延迟队列的消息：{}", new Date().toString(), msg);
}
```

发送两条消息，这个时候我们可以看到低延迟的先输出：

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312214738.png)

## 4.总结

RabbitMQ 来实现延时队列可以很好的利用 RabbitMQ 的特性，如：消息可靠发送、消息可靠投递、死信队列来保障消息至少被消费一次以及未被正确处理的消息不会被丢弃。另外，通过 RabbitMQ 集群的特性，可以很好的解决单点故障问题，不会因为单个节点挂掉导致延时队列不可用或者消息丢失。
