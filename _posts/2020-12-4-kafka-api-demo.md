---
layout: post
title: kafka
keywords: kafka
description: kafka
categories: [message queue, kafka]
tags: [message queue, kafka]
---

# kafka producer
    
    Properties properties = new Properties();
    // ProducerConfig 类定义了生产者相关的配置
    // 指定kafka 连接的集群
    properties.put("bootstrap.servers", "xxx");
    // ack应答级别
    properties.put("acks", "all");
    // 重试次数
    properties.put("retries", 0);
    // 单次发送批次的大小
    properties.put("batch.size", 16384);
    // 批次大小不足的情况等待发送时间
    properties.put("linger.ms", 1);
    // RecordAccumulator 缓冲区大小
    properties.put("buffer.memory", 33554432);
    // key - value序列化
    properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");    
    
    
    Producer<String, String> producer = new KafkaProducer<String, String>(properties);
    ProducerRecord<String, String> record = new ProducerRecord<>(“Kafka”, “Kafka_Products”, “测试”);//Topic Key Value
    producer.send(record)
    
# kafka consumer

    bootstrap.servers： kafka的地址。
    
    group.id：组名 不同组名可以重复消费。例如你先使用了组名A消费了kafka的1000条数据，但是你还想再次进行消费这1000条数据，并且不想重新去产生，那么这里你只需要更改组名就可以重复消费了。
    
    enable.auto.commit：是否自动提交，默认为true。
    
    auto.commit.interval.ms: 从poll(拉)的回话处理时长。
    
    session.timeout.ms:超时时间。
    
    max.poll.records:一次最大拉取的条数。
    
    auto.offset.reset：消费规则，默认earliest 。
    
        earliest: 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费 。
        latest: 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据 。
        none: topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常。
    
    key.serializer: 键序列化，默认org.apache.kafka.common.serialization.StringDeserializer。
    
    value.deserializer:值序列化，默认org.apache.kafka.common.serialization.StringDeserializer
    
    
    Properties props = new Properties();
    props.put("bootstrap.servers", "master:9092,slave1:9092,slave2:9092");
    props.put("group.id", GROUPID);
    props.put("enable.auto.commit", "true");
    props.put("auto.commit.interval.ms", "1000");
    props.put("session.timeout.ms", "30000");
    props.put("max.poll.records", 1000);
    props.put("auto.offset.reset", "earliest");
    props.put("key.deserializer", StringDeserializer.class.getName());
    props.put("value.deserializer", StringDeserializer.class.getName());
    KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
    
    consumer.subscribe(Arrays.asList(topic));
    ConsumerRecords<String, String> msgList=consumer.poll(1000);

# kafka stream
提供了对存储于Kafka内的数据进行流式处理和分析的功能

## KStream和KTable

    Kstream：
    1.Kstream是一个由键值对构成的抽象记录流，每个键值对是一个独立的单元，即使相同的Key也不会覆盖，类似数据库的插入操作；
    
    2.Ktable：相同Key的每条记录只保存最新的一条记录，类似于数据库的基于主键更新
    
    无论是记录流（KStream）还是更新日志流（KTable）都可以从一个或者多个Kafka主题数据源来创建，一个Kstream可以与另一个Kstream或者Ktable进行join操作，或者聚合成一个Ktable，同样一个Ktable也可以转换成一个Kstream。
    
    Properties props = new Properties();
    //设置程序的唯一标识
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount-application");
    //设置kafka集群
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "node01:9092,node02:9092,node03:9092");
    //设置序列化与反序列化
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
 
    //实例一个计算逻辑
    StreamsBuilder streamsBuilder = new StreamsBuilder();
    //设置计算逻辑   stream：读取                                                         to：写入
    streamsBuilder.stream("18BD34").mapValues(line->line.toString().toUpperCase()).to("18BD34-1");
    
    //构建Topology对象（拓扑，流程）
    final Topology topology = streamsBuilder.build();
    //实例 kafka流
    KafkaStreams streams = new KafkaStreams(topology, props);
    //启动流计算
    streams.start();
    
开发者可以通过Processor接口来实现自己的自定义处理逻辑。接口提供了Process和Punctuate方法。

其中：Process方法用于处理接受到的消息

Punctuate方法指定时间间隔周期性的执行，用于处理周期数据：例如某些状态值计算生成新的流。

Processor接口还提供了init方法，init初始化方法可以将ProcessorContext转存到Procesor实例中，以供Prounctute使用。

可以使用context的schedule方法实现punctute的周期性调用。

将修改后的数据转存到下游处理节点：context.().forward

体检当前处理节点的处理进度：context.commit.  
  
    public class MyProcessor extends Processor {
 
        private ProcessorContext context;
 
        private KeyValueStore kvStore;
        @Override
        @SuppressWarnings("unchecked")
        public void init(ProcessorContext context) {
            this.context = context;
            this.context.schedule(1000);
            this.kvStore = (KeyValueStore) context.getStateStore("Counts");
        }
        @Override
        public void process(String dummy, String line) {
            String[] words = line.toLowerCase().split(" ");
            for (String word : words) {
                Integer oldValue = this.kvStore.get(word);
                if (oldValue == null) {
                    this.kvStore.put(word, 1);
                } else {
                    this.kvStore.put(word, oldValue + 1);
                }
            }
        }
        @Override
        public void punctuate(long timestamp) {
            KeyValueIterator iter = this.kvStore.all();
            while (iter.hasNext()) {
                KeyValue entry = iter.next();
                context.forward(entry.key, entry.value.toString());
            }
            iter.close();
            context.commit();
        }
        @Override
        public void close() {
            this.kvStore.close();
        }
    };
    
在上面的代码中：

1、 init方法，定义了每秒调用punctuate方法，将名称为count的状态存储结构中转存到奔processor处理节点中。

2、 在process方法中，每接受到一条消息，将字符串进行拆分，并更新到状态存储中，生成新的流。

3、 在puncuate方法中，迭代本地状态存储并将流提交到下个处理节点进行处理。