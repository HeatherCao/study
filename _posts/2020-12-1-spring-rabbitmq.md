---
layout: post
title: springboot rabbitmq
keywords: rabbitmq
description: rabbitmq
categories: [springboot, rabbitmq]
tags: [springboot, rabbitmq]
---

# rabbitmq如何解决消息重复消费以及幂等性问题？
    首先将 RabbitMQ 的消息自动确认机制改为手动确认，然后每当有一条消息消费成功了，就把该消息的唯一 ID 记录在
    Redis 上，然后每次收到消息时，都先去 Redis 上查看是否有该消息的 ID，如果有，表示该消息已经消费过了，不再处理，否则再去处理
    
# 多个消费者监听一个队列时，消息如何分发？
    1.Round-Robin（轮询）
    默认的策略，消费者轮流，平均地收到消息。
    
    2.Fair dispatch（公平分发）
    如果要实现根据消费者的处理能力来分发消息，给空闲的消费者发送更多消息，可以用basicQos(int prefetch_count)来设置。  
    prefetch_count的含义：当消费者有多少条消息没有响应ACK时，不再给这个消费者发送消息。
    
    /**
     * basicQos(int prefetchCount)
     * prefetchCount:maximum number of messages that the server will deliver, 0 if unlimited
    */
    channel.basicQos(1);//阻止rabbitmq将消息平均分配到每一个消费者，会优先的发给不忙的消费者，如果当前的消费者在忙的话，就将消息分配给下一个消费者

# rabbitmq如何处理无法被路由的消息？
如果没有任何设置，无法路由的消息会被直接丢弃。
无法路由的情况：Routing key不正确
解决方案：
    
1.使用mandatory = true配合ReturnListener(Spring AMQP中是returnCallBack)，实现消息回发
2.消息路由到备份交换机的方式：在创建交换机的时候，从属性中指定备份交换机。
AlternateExchange（AE）声明交换机的时候添加属性alternate-exchange，声明一个备用交换机，一般声明为fanout类型，这样交换机收到路由不到队列的消息就会发送到备用交换机绑定的队列中。
    
    Map<String, Object> arguments = new HashMap<>(16);
    arguments.put("alternate-exchange", "backup-exchange");
    channel.exchangeDeclare(EXCHANGE_NAME, "direct", true, false, arguments);
    channel.queueDeclare(QUEUE_NAME, true, false, false, null);
    channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, BINDING_KEY);
     
    // 声明一个 fanout 类型的交换器，建议此处使用 fanout 类型的交换器
    channel.exchangeDeclare("backup-exchange", "fanout", true, false, null);
    // 消息没有被路由的之后存入的队列
    channel.queueDeclare("unRoutingQueue", true, false, false, null);
    channel.queueBind("unRoutingQueue", "backup-exchange", "");
     
    // 发送一条持久化消息
    String message = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) + " 没有被正确的路由到消息队列，此时此消息会进入 unRoutingQueue";
    try {
    // 使用 routingKey
    channel.basicPublish(EXCHANGE_NAME, "not-exists-routing-key", true, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes(StandardCharsets.UTF_8));
    System.err.println("消息发送完成......");
    } catch (IOException e) {
    	e.printStackTrace();
    }

# rabbitmq死信队列
rabbitmq中有种交换机叫DLX，全称Data-Letter-Exchange, 可以称之为死信交换机。  
当消息在一个队列中变成死信时，它就会被重新发送到设置的另一个交换机，这个交换机我们就称之为死信交换机DLX，绑定在DLX上的队列就是死信队列。

消息变为死信队列一般是以下原因造成的：
    
    1.消息被拒(basic.reject or basic.nack)并且没有重新入队(requeue=false)
    2.消息在队列中过期，即当前消息在队列中的存活时间已经超过了预先设置的TTL(Time To Live)时间
    3.当前队列中的消息数量已经超过最大长度

注意，并不是直接声明一个公共的死信队列，然后所以死信消息就自己跑到死信队列里去了。   
而是为每个需要使用死信的业务队列配置一个死信交换机，这里同一个项目的死信交换机可以共用一个，然后为每个业务队列分配一个单独的路由key。

注意，并不是直接声明一个公共的死信队列，然后所以死信消息就自己跑到死信队列里去了。而是为每个需要使用死信的业务队列配置一个死信交换机，这里同一个项目的死信交换机可以共用一个，然后为每个业务队列分配一个单独的路由key。

    @Configuration
    public class RabbitMQConfig {
        public static final String BUSINESS_EXCHANGE_NAME = "dead.letter.demo.simple.business.exchange";
        public static final String BUSINESS_QUEUEA_NAME = "dead.letter.demo.simple.business.queuea";
        public static final String BUSINESS_QUEUEB_NAME = "dead.letter.demo.simple.business.queueb";
        public static final String DEAD_LETTER_EXCHANGE = "dead.letter.demo.simple.deadletter.exchange";
        public static final String DEAD_LETTER_QUEUEA_ROUTING_KEY = "dead.letter.demo.simple.deadletter.queuea.routingkey";
        public static final String DEAD_LETTER_QUEUEB_ROUTING_KEY = "dead.letter.demo.simple.deadletter.queueb.routingkey";
        public static final String DEAD_LETTER_QUEUEA_NAME = "dead.letter.demo.simple.deadletter.queuea";
        public static final String DEAD_LETTER_QUEUEB_NAME = "dead.letter.demo.simple.deadletter.queueb";
    
        // 声明业务Exchange
        @Bean("businessExchange")
        public FanoutExchange businessExchange(){
            return new FanoutExchange(BUSINESS_EXCHANGE_NAME);
        }
    
        // 声明死信Exchange
        @Bean("deadLetterExchange")
        public DirectExchange deadLetterExchange(){
            return new DirectExchange(DEAD_LETTER_EXCHANGE);
        }
    
        // 声明业务队列A
        @Bean("businessQueueA")
        public Queue businessQueueA(){
            Map<String, Object> args = new HashMap<>(2);
    //       x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
            args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
    //       x-dead-letter-routing-key  这里声明当前队列的死信路由key
            args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEA_ROUTING_KEY);
            return QueueBuilder.durable(BUSINESS_QUEUEA_NAME).withArguments(args).build();
        }
    
        // 声明业务队列B
        @Bean("businessQueueB")
        public Queue businessQueueB(){
            Map<String, Object> args = new HashMap<>(2);
    //       x-dead-letter-exchange    这里声明当前队列绑定的死信交换机
            args.put("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE);
    //       x-dead-letter-routing-key  这里声明当前队列的死信路由key
            args.put("x-dead-letter-routing-key", DEAD_LETTER_QUEUEB_ROUTING_KEY);
            return QueueBuilder.durable(BUSINESS_QUEUEB_NAME).withArguments(args).build();
        }
    
        // 声明死信队列A
        @Bean("deadLetterQueueA")
        public Queue deadLetterQueueA(){
            return new Queue(DEAD_LETTER_QUEUEA_NAME);
        }
    
        // 声明死信队列B
        @Bean("deadLetterQueueB")
        public Queue deadLetterQueueB(){
            return new Queue(DEAD_LETTER_QUEUEB_NAME);
        }
    
        // 声明业务队列A绑定关系
        @Bean
        public Binding businessBindingA(@Qualifier("businessQueueA") Queue queue,
                                        @Qualifier("businessExchange") FanoutExchange exchange){
            return BindingBuilder.bind(queue).to(exchange);
        }
    
        // 声明业务队列B绑定关系
        @Bean
        public Binding businessBindingB(@Qualifier("businessQueueB") Queue queue,
                                        @Qualifier("businessExchange") FanoutExchange exchange){
            return BindingBuilder.bind(queue).to(exchange);
        }
    
        // 声明死信队列A绑定关系
        @Bean
        public Binding deadLetterBindingA(@Qualifier("deadLetterQueueA") Queue queue,
                                        @Qualifier("deadLetterExchange") DirectExchange exchange){
            return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEA_ROUTING_KEY);
        }
    
        // 声明死信队列B绑定关系
        @Bean
        public Binding deadLetterBindingB(@Qualifier("deadLetterQueueB") Queue queue,
                                          @Qualifier("deadLetterExchange") DirectExchange exchange){
            return BindingBuilder.bind(queue).to(exchange).with(DEAD_LETTER_QUEUEB_ROUTING_KEY);
        }
    }
    
以上代码声明了两个Exchange，一个是业务Exchange，另一个是死信Exchange，业务Exchange下绑定了两个业务队列，业务队列都配置了同一个死信Exchange，并分别配置了路由key，在死信Exchange下绑定了两个死信队列，设置的路由key分别为业务队列里配置的路由key。

配置文件需要修改：

    //  (若有重试)重试次数超过限制后】,是否将被拒绝的消息重新入队。
    spring.rabbitmq.listener.simple.default-requeue-rejected
    
# rabbitmq如何实现延迟队列
延迟队列存储的对象是对应的延迟消息，所谓"延迟消息"是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。   

在AMQP协议中，或者RabbitMQ本身没有直接支持延迟队列的功能，但是可以通过配置死信队列和设置消息或队列的过期时间来模拟出延迟队列的功能。

使用场景
    
    1.在订单系统中，用户下单之后通常有30分钟的时间进行支付，如果30分钟之内没有支付成功，那么这个订单将进行异常处理，这时就可以使用延迟队列来处理这些订单了
    2.用户希望通过手机远程遥控家里的智能设备在指定的时间进行工作。这时候就可以将用户指令发送到延迟队列，当指令设定的时间到了再将指令推送到智能设备
    
设置消息的过期时间
![](/images/2020-12-1-spring-rabbitmq.png)

设置队列的过期时间

      @Bean
      public Queue TTLQueue() {
          Map<String, Object> map = new HashMap<>();
          // 队列中的消息未被消费则30秒后过期
          map.put("x-message-ttl", 30000); 
          return new Queue("TTL_QUEUE", true, false, false, map);
      }
      
# SpringBoot中，bean还没有初始化好，消费者就开始监听取消息，导致空指针异常，怎么让消费者在容器启动完毕后才开始监听？

1.RabbitMQ中有一个auto_startup参数，可以控制是否在容器启动时就监听启动
    
    spring.rabbitmq.listener.auto-startup=true   ##默认是true

2.自定义容器，容器可以应用到消费者：

    // 默认是true 
    factory.setAutoStartup(true);
    
3.消费者单独设置

    @RabbitListener( queues = "${com.gupaoedu.thirdqueue}" ,autoStartup = "true")

