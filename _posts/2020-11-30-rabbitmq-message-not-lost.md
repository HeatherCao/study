---
layout: post
title: springboot rabbitmq
keywords: rabbitmq
description: rabbitmq
categories: [springboot, rabbitmq]
tags: [springboot, rabbitmq]
---
#rabbitmq怎么保证消息的可靠性传输？

##①生产者丢失数据
###1.开启消息确认机制
    增加配置
    // 开启消息发送到交换机的确认机制
    spring.rabbitmq.publisher-confirms = true
    // 开启消息从交换机发送到队列的确认机制
    spring.rabbitmq.publisher-returns = true
    // 开启强制消息投递，消息如果未被路由至任何一个队列，则回退一条消息到rabbitTemplate.returnCallback中
    spring.rabbitmq.template.mandatory = true
    
#### ConfirmCallback确认模式
    @Slf4j
    @Component
    public class ConfirmCallbackService implements RabbitTemplate.ConfirmCallback {
        @Override
        public void confirm(CorrelationData correlationData, boolean ack, String cause) {
            if (!ack) {
                log.error("消息发送异常!");
            } else {
                log.info("发送者爸爸已经收到确认，correlationData={} ,ack={}, cause={}", correlationData.getId(), ack, cause);
            }
    }
实现接口 ConfirmCallback ，重写其confirm()方法，方法内有三个参数correlationData、ack、cause。  
correlationData：对象内部只有一个 id 属性，用来表示当前消息的唯一性。  
ack：消息投递到broker 的状态，true表示成功。    
cause：表示投递失败的原因。
##### 从2.1版本开始，CorrelationData对象具有ListenableFuture功能，可以获取结果，代替ConfirmCallBack
    CorrelationData cd1 = new CorrelationData();
    this.templateWithConfirmsEnabled.convertAndSend("exchange", queue.getName(),"foo",cd1);
    assertTure(cd1.getFuture.get(10, TimeUnit.SECONDS).isAck());
 
#### ReturnCallBack确认模式
如果消息未能投递到目标 queue 里将触发回调 returnCallback ，一旦向 queue 投递消息未成功，这里一般会记录下当前消息的详细投递数据，方便后续做重发或者补偿等操作。
    
    @Slf4j
    @Component
    public class ReturnCallbackService implements RabbitTemplate.ReturnCallback {
        @Override
        public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
            log.info("returnedMessage ===> replyCode={} ,replyText={} ,exchange={} ,routingKey={}", replyCode, replyText, exchange, routingKey);
        }
    }
实现接口ReturnCallback，重写 returnedMessage() 方法，方法有五个参数message（消息体）、replyCode（响应code）、replyText（响应内容）、exchange（交换机）、routingKey（队列）。

#### 具体的消息发送示例
    @Autowired
    private RabbitTemplate rabbitTemplate;
 
    @Autowired
    private ConfirmCallbackService confirmCallbackService;
 
    @Autowired
    private ReturnCallbackService returnCallbackService;
 
    public void sendMessage(String exchange, String routingKey, Object msg) {
        // 确保消息发送失败后可以重新返回到队列中
        // 注意：yml需要配置 publisher-returns: true
        rabbitTemplate.setMandatory(true);
        
        // 消费者确认收到消息后，手动ack回执回调处理
        rabbitTemplate.setConfirmCallback(confirmCallbackService);
 
        // 消息投递到队列失败回调处理
        rabbitTemplate.setReturnCallback(returnCallbackService);
 
        // 发送消息
        rabbitTemplate.convertAndSend(exchange, routingKey, msg,
                message -> {
                    // 消息持久化，MQ持久化是将消息记录到磁盘，对性能有影响
                    message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                    return message;
                },
                new CorrelationData(UUID.randomUUID().toString()));
    }    
    
### 2.开启事务机制
事务机制是同步的（不推荐，同步操作，会阻塞，效率低下），而确认机制是异步的。
spring.rabbitmq.publisher-confirms一定要配置为false，否则会与事务处理相冲突，启动时会报异常。

    // 配置启用rabbitmq事务
    @Bean("rabbitTransactionManager")
        public RabbitTransactionManager rabbitTransactionManager(CachingConnectionFactory connectionFactory) {
    	    return new RabbitTransactionManager(connectionFactory);
    	}
    }   

    /**
	 * RabbitMQ消息发送类
	 */
	@Component
	public class RabbitSender implements RabbitTemplate.ReturnCallback {
	 
		private static final Logger logger = LoggerFactory.getLogger("mq");
		
	    @Autowired
	    private RabbitTemplate rabbitTemplate;
	    
		@Value("${mq.exchange}")
		private String exchange;
		
		@Value("${mq.ingate.queue}")
		private String ingateQueue;
	    
		@PostConstruct
		private void init() {
		    //启用事务模式,不能开确认回调
			//rabbitTemplate.setConfirmCallback(this);
	        rabbitTemplate.setReturnCallback(this);
	        //激活rabbitTemplate对象事务处理功能
			rabbitTemplate.setChannelTransacted(true);
			rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
		}
		
		@Transactional(rollbackFor = Exception.class,transactionManager = "rabbitTransactionManager")
	    public void sendIngateQueue(TradePayModelRes msg) {
			logger.info("进闸消息已发送 {}",msg.getOutTradeNo());
			rabbitTemplate.convertAndSend(exchange,ingateQueue,msg);
	    }

	    @Override
	    public void returnedMessage(Message message, int replyCode, String replyText,String exchange, String routingKey) {
	    	logger.info("消息被退回 {}" , message.toString());
	    }
	}
本类用于统一向RabbitMQ发送消息，在发送消息的sendIngateQueue()方法上，加了@Transactional注解，表示这个方法将启用事务（此时的事务即是RabbitMQ事务，因为前面定义了RabbitTransactionManager ）。  
由于启用了事务，所以需要在系统初始化时，调用rabbitTemplate.setChannelTransacted(true)，以激活rabbitTemplate对象事务处理功能。

## ②消息队列丢失数据
消息队列在很多场景下都是需要有持久下机制了。否则如果队列中挤压了大量消息，此时mq服务器重启，那么这些消息就都消失了，因为这些消息都只存在于内存，而RabbitMQ为了解决这个问腿，也提供了持久化机制
### 1.交换机持久化
交换机持久化描述的是当这个交换机上没有注册队列时，这个交换机是否删除。如果要打开持久化的话也很简单

    @Bean
    public DirectExchange testDirectExchange(){
         //第二个参数就是是否持久化，第三个参数就是是否自动删除
         return new DirectExchange("direct.Exchange",true,false);
    }
消费者类上的注解：

    @RabbitListener(
            bindings = @QueueBinding(
                    value = @Queue(value = "direct.Queue",autoDelete = "true"),
                    exchange = @Exchange(value = "direct.Exchange", type = ExchangeTypes.DIRECT,durable = "true"),
                    key = "direct.Rout"
            )
    )
@Exchange注解的durable属性设置为true(默认也是true，不设置也可以)。这样，即使这个交换机没有队列，也不会被删除。 exchange持久化存储  
当autoDelete属性设置到该注解的时候，含义即是，当所有绑定队列都不在使用时，是否自动删除交换器，当设置值是true的时候删除该交换器，当值是false的时候不删除该交换器。

### 2.队列持久化
    @Bean
    public Queue txQueue(){
        //第二个参数就是durable，是否持久化
        return new Queue("txQueue",true);
    }
    
    @RabbitListener(
            bindings = @QueueBinding(
                    value = @Queue(value = "direct.Queue",autoDelete = "false",durable = "true"),
                    exchange = @Exchange(value = "direct.Exchange", type = ExchangeTypes.DIRECT,durable = "true"),
                    key = "direct.Rout"
            )
    )
    可以看到，@Queue注解上，加上了durable="true"的注解。这样队列在重启的时候就不会被删除了
    当autoDelete属性设置到该注解的时候，含义即是，当所有消费者客户端连接断开后，是否自动删除队列，当设置值是true的时候删除该队列，当值是false的时候不删除该队列。

### 3.消息持久化
     rabbitTemplate.convertAndSend(exchange, routingKey, msg,
                    message -> {
                        // 消息持久化，MQ持久化是将消息记录到磁盘，对性能有影响
                        message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                        return message;
                    }
                    
## ③消费者丢失数据
启用手动确认模式可以解决这个问题

1.NONE：无应答，rabbitmq默认consumer正确处理所有请求。  
2.AUTO：consumer自动应答，处理成功（注意：此处的成功确认是没有发生异常）发出ack，处理失败发出nack。rabbitmq发出消息后会等待consumer端应答，只有收到ack确定信息后才会将消息在rabbitmq清除掉。收到nack异常信息的处理方法由setDefaultRequeueReject()方法设置，这种模式下，发送错误的消息可以恢复。  
3.MANUAL：基本等同于AUTO模式，区别是需要人为调用方法确认。

接口层接受到一个请求，然后推送一个消息，异步地去更新数据库。此时对于消费者端来说，一拿到消息，消息就从队列上被删除，然后开始执行数据库更新，但此时数据库更新失败了，方法直接返回。但队列上已经没有这条消息了，这个更新操作不就没有完成了吗？这肯定是有问题的。
所以RabbitMQ就有了消费者确认机制，只有消费者手动确认，消息才会被删除，否则该消息将一直存在队列中，开启的方法很简单：

    spring.rabbitmq.listener.simple.acknowledge-mode=manual
    
对于消费者端：           

    // 手动调用下确认即可，消息就会被删除。这一步可以放在业务逻辑的执行之后
    @RabbitHandler
    public void onMessage(String str,Channel channel,Message message) throws IOException {
        channel.basicAck(message.getMessageProperties().getDeliveryTag(),false); //手动调用
        System.out.println(str);
    }
    
    // 经过开发中的实际测试，当消息回滚到消息队列时，这条消息不会回到队列尾部，而是仍是在队列头部，这时消费者会立马又接收到这条消息进行处理，接着抛出异常，进行回滚，如此反复进行。这种情况会导致消息队列处理出现阻塞，消息堆积，导致正常消息也无法运行。对于消息回滚到消息队列，我们希望比较理想的方式时出现异常的消息到达消息队列尾部，这样既保证消息不会丢失，又保证了正常业务的进行，因此我们采取的解决方案是，将消息进行应答，这时消息队列会删除该消息，同时我们再次发送该消息到消息队列，这时就实现了错误消息进行消息队列尾部的方案。
    // 重新发送消息到队尾
    channel.basicPublish(message.getMessageProperties().getReceivedExchange(),
          message.getMessageProperties().getReceivedRoutingKey(), MessageProperties.PERSISTENT_TEXT_PLAIN,
          JSON.toJSONBytes(new Object()));
    
如果设置了重试模式，那么在出现异常时没有捕获异常会进行重试，如果捕获了异常不会重试。      

    spring.rabbitmq.listener.simple.retry.max-attempts=5  //最大重试次数
    spring.rabbitmq.listener.simple.retry.enabled=true //是否开启消费者重试（为false时关闭消费者重试，这时消费端代码异常会一直重复收到消息）
    spring.rabbitmq.listener.simple.retry.initial-interval=5000 //重试间隔时间（单位毫秒）
    spring.rabbitmq.listener.simple.default-requeue-rejected=false //重试次数超过上面的设置之后是否丢弃（false不丢弃时需要写相应代码将该消息加入死信队列）

# rabbitmq消息基于什么传输？

由于TCP连接的创建和销毁开销较大，且并发数受系统资源限制，会造成性能瓶颈。

    RabbitMQ使用信道的方式来传输数据。信道是建立在真实的TCP连接内的虚拟连接，且每条TCP连接上的信道数量没有限制。
    
    