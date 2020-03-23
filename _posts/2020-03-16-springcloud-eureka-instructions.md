---
layout: post
title: springcloud组件之一——Eureka的使用
keywords: Eureka
description: eureka应用入门
categories: hi
tags: [springCloud, Eureka]
---

*  目录
{:toc}

# Eureka入门
eureka分为服务端和客户端
## 搭建eureka服务端
三板斧：

①加依赖
 
    <dependency> 
        <groupId>org.springframework.cloud</groupId> 
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId> 
    </dependency>
②:写注解 在主启动类上 写上@EnableEurekaServer注解

    @EnableEurekaServer 
    public class EurekaServerApplication {
    
    }
③:编写配置文件
    
    #eureka服务端应用的端口默认是8761 
    server.port=9000 
    #表示是否将自己注册到Eureka Server,默认为true,由于当前应用就是Eureka Server,故而设为false 
    eureka.client.register-with-eureka=false 
    # 表示是否从Eureka Server获取注册信息,默认为true,因为这是一个单点的Eureka Server,不需要同步其他的Eureka Server节点的数据,故而设为false 
    eureka.client.fetch-registry=false 
    #暴露给其他eureka client 的注册地址 
    eureka.client.service-url.defaultZone: http://localhost:9000/eureka/
## 搭建eureka客户端
①：加依赖

    <dependency> 
        <groupId>org.springframework.cloud</groupId> 
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId> 
    </dependency>
②：加入注解@EnableDiscoveryClient

    @EnableDiscoveryClient 
    public class ConsumerUserApplication {
    
    }
③：编写配置文件

    server.port=8001 
    #注册到eureka服务端的微服务名称 
    spring.application.name=ms-consumer-user 
    #注册到eureka服务端的地址 
    eureka.client.service-url.defaultZone=http://localhost:9000/eureka/ 
    #点击具体的微服务，右下角是否显示ip 
    eureka.instance.prefer-ip-address=true 
    #显示微服务的名称 
    eureka.instance.instance-id=ms-consumer-user-8001
    
④: 调用 使用RestTemplate （调用的时候不再是通过Ip 和地址发起调用，而是通过微服务名称进行调用） 

需要加入@LoadBanlance 注解

    @RequestMapping("/queryUserInfoById/{userId}") 
    public UserInfoVo queryUserInfoById(@PathVariable("userId") Integer userId) { 
        User user = userServiceImpl.queryUserById(userId); 
        ResponseEntity<List> responseEntity = restTemplate.getForEntity("http://MS-PROVIDER-ORDER/order/queryOrdersByUserId/"+userId,List.class); 
        List<OrderVo> orderVoList = responseEntity.getBody(); 
        UserInfoVo userInfoVo = new UserInfoVo(); 
        userInfoVo.setOrderVoList(orderVoList); 
        userInfoVo.setUserName(user.getUserName()); 
        return userInfoVo; 
    }
    
    @Configuration public class MainConfig { 
        @Bean 
        @LoadBalanced 
        public RestTemplate restTemplate() { 
            return new RestTemplate(); 
        } 
    }
![](/images/2020-03-16-springcloud-eureka-instructions-01.png)

## Eureka 部署架构
![](/images/2020-03-16-springcloud-eureka-instructions-02.png)

Region(us-east-1,) 可以类比于 阿里云的不同区域

Region 之间的内网是不相通的

Availabitlity zone 可用区(us-east-1c us-east-1d us-east-1e) 可以理解为 我们 一个地区比如华南地区，不 同城市的机房，内网是相通的

1.Eureka Server提供服务发现的能力，各个微服务启动时，会向Eureka Server注册自己的信息（例 如IP、端口、微服务名称等），Eureka Server会存储这些信息；

2.Eureka Client是一个Java客户端，用于简化与Eureka Server的交互(启动的时候，会向server注册自己的 信息);

3.微服务启动后 renew，会周期性（默认30秒）地向Eureka Server发送心跳以续约自己的“租期”；

4.如果Eureka Server在一定时间内没有接收到某个微服务实例的心跳（最后一次续约时间开始计 算），Eureka Server将会注销该实例（默认90秒）；

5.默认情况下，Eureka Server同时也是Eureka Client。多个Eureka Server实例，互相之间通过增量复制的方 式，来实现服务注册表中数据的同步。Eureka Server默认保证在90秒内，Eureka Server集群内的所有 实例中的数据达到一致（从这个架构来看，Eureka Server所有实例所处的角色都是对等的，没有类 似Zookeeper、选举过程，也不存在主从，所有的节点都是主节点。Eureka官方将Eureka Server集 群中的所有实例称为“对等体（peer）”）;

6.Eureka Client会缓存服务注册表中的信息。这种方式有一定的优势——首先，微服务无需每次请求都 查询Eureka Server，从而降低了Eureka Server的压力；其次，即使Eureka Server所有节点都宕 掉，服务消费者依然可以使用缓存中的信息找到服务提供者并完成调用。

## Eureka配置的总要信息讲解
1.服务注册配置 

eureka.client.register-with-eureka=true

该项配置说明,是否向注册中心注册自己,在非集群环境下设置为false，表示自己不注册自己

eureka.client.fetch-registry=true 

该项配置说明，注册中心只要维护微服务实例清单，非集群环境下,不需要作检索服务,也设置为false

2.服务续约配置 

eureka.instance.lease-renewal-interval-in-seconds=30(默认) 

该配置说明，服务提供者会维持一个心跳告诉eureka server 我还活着，这个就是一个心跳周期

eureka.instance.lease-expiration-duration-in-seconds=90(默认) 

该配置说明,最后一次续约时间开始，往后推90s 还没接受到心跳,那么就需要剔除.

3.获取服务配置 (前提,eureka.client.fetch-registry为true) 

eureka.client.registry-fetch-interval-seconds=30 

缓存在调用方的微服务实例清单刷新时间

4.eureka 的自我保护功能

![](/images/2020-03-16-springcloud-eureka-instructions-03.png)

默认情况下，若eureka server 在一段时间内（90s）没有接受到某个微服务实例的心跳，那么 eureka server 就会剔除 该实例，当时由于网络分区故障，导致eureka server 和 服务之间无法通信，---
因为微服务是实例健康的,本不应注销该实例. 

那么通过eureka server 自我保护模式就起作用了,当eureka server节点短时间(15min是否低于85%)丢失过多客户端 是(由于网络分区)，那么该eureka server节点就会进入自我保护模式，eureka server 就会自动保护注册表中的微服务 实例，不再删除该注册表中微服务的信息，等到网络恢复，eureka server 节点就会退出自我保护功能(宁可放过一千， 不要错杀一个)

## Eureka导入安全配置
1.导入安全配置 依赖

    <dependency> 
        <groupId>org.springframework.boot</groupId> 
        <artifactId>spring-boot-starter-security</artifactId> 
    </dependency>
    
2.修改yml配置文件

//注册的时候需要带入 用户名 密码 基本格式为 

http:用户名:密码@ip:端口/eureka/ 

    eureka.client.service-url.defaultZone: http://${spring.security.user.name}:${spring.security.user.name}@www.eureka9000.com:9000/eureka/
    //开启默认的认证 
    spring.security.basic.enable=true 
    spring.security.user.name=root 
    spring.security.user.password=123456
    
=====================客户端配置=======================

注册地址修改为这个 

eureka.client.service-url.defaultZone=http://${security.login.username}:${security.login.pass}@www.eureka9000.com:9000/eureka/

![](/images/2020-03-16-springcloud-eureka-instructions-04.png)
