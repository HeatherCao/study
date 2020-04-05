---
layout: post
title: springcloud组件——Ribbon
keywords: Ribbon
description: Ribbon
categories: [springCloud, Ribbon]
tags: [springCloud, Ribbon]
---

*  目录
{:toc}

# Ribbon
## 什么是Ribbon

    spring cloud ribbon是 基于NetFilix ribbon 实现的一套客户端的负载均衡工具,Ribbon客户端组件提供一系列的完善的配置，如超时，重试等。
    通过Load Balancer（LB）获取到服务提供的所有机器实例，Ribbon会自动基于某种规则(轮询，随机)去调用这些服务。Ribbon也可以实现
    我们自己的负载均衡算法。
### 什么是客户端的负载均衡

    进程内的LB,他是一个类库集成到消费端，通过消费端进行获取提供者的地址
    生活中:类似与你去火车站排队进站（有三条通道），只要是正常人，都会排队到人少的队伍中去.
    程序中: 我们消费端 能获取到服务提供者地址列表，然后根据某种策略去获取一个地址进行调用.
    
![](/images/2020-04-05-springcloud-ribbon-instructions-01.png)

### 什么是服务端的负载均衡

    生活中:类似与你去火车站排队进站的时候，有一个火车站的引导员告诉你说三号通道人少，你去三号通道排队。
    程序中:就是你消费者调用的是ng的地址，由ng来选择 请求的转发(轮询 权重，ip_hash等)

![](/images/2020-04-05-springcloud-ribbon-instructions-02.png)

## 如何快速整合ribbon

    核心：
    把ribbon集成到消费端
    修改调用配置
    
    public class MainConfig {
        @Bean
        @LoadBalanced
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }
    }

    @RequestMapping("/queryUserInfoById/{userId}")
    public UserInfoVo queryUserInfoById(@PathVariable("userId") Integer userId) {
        User user = userServiceImpl.queryUserById(userId);
        //通过微服务实例名称进行调用
        ResponseEntity<List> responseEntity = restTemplate.getForEntity("http://MS-PROVIDER-ORDER/order/ queryOrdersByUserId/"+userId,List.class);
        //ResponseEntity<List> responseEntity = restTemplate.getForEntity("http://localhost:8002/order/queryOrdersByUserId/"+userId,List.class);
        List<OrderVo> orderVoList = responseEntity.getBody();

        UserInfoVo userInfoVo = new UserInfoVo();
        userInfoVo.setOrderVoList(orderVoList);
        userInfoVo.setUserName(user.getUserName());
        return userInfoVo;
    }
    
## ribbon核心知识点
### Ribbon所包含的负载均衡策略分类

    1.RandomRule：随机选择一个Server
    2.RetryRule：对选定的负载均衡策略机上重试机制，在一个配置时间段内当选择Server不成功，则一直尝试使用subRule的方式选择一个可用的server
    3.RoundRobinRule：轮询选择， 轮询index，选择index对应位置的Server
    4.AvailabilityFilteringRule：过滤掉一直连接失败的被标记为circuit tripped的后端Server，并过滤掉那些高并发的后端Server或者使用一个AvailabilityPredicate来包含过滤server的逻辑，其实就就是检查status里记录的各个Server的运行状态
    5.BestAvailableRule：选择一个最小的并发请求的Server，逐个考察Server，如果Server被tripped了，则跳过。
    6.WeightedResponseTimeRule：根据响应时间加权，响应时间越长，权重越小，被选中的可能性越低；
    7.ZoneAvoidanceRule：复合判断Server所在区域的性能和Server的可用性选择Server

### 如何修改默认负载均衡的策略

    写一个配置类来修改ribbon的负载均衡的配置
    @Configuration
    public class MainConfig {
        @Bean
        @LoadBalanced
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }
    
        /**
         * 随机算法的负载均衡
         * @return
         */
        @Bean
        public IRule TulingRule() {
            //return new RandomRule();
            return new RetryRule();
        }
    }

### 如何自定义负载均衡策略
#### 1.引用规则,使用@RibbonClient指定负载均衡策略

    @SpringBootApplication
    @EnableDiscoveryClient
    @RibbonClient(name = "MS-PROVIDER-ORDER",configuration = TulingRandomRule.class)
    public class Tulingvip02MsConsumerUser8001Application{} 
    
#### 2.自定义的负载均衡策略不能写在@SpringbootApplication注解的@CompentScan扫描得到的地方

#### 3.自定义负载均衡策略

    package com.config;
    import com.netflix.client.config.IClientConfig;
    import com.netflix.loadbalancer.AbstractLoadBalancerRule;
    import com.netflix.loadbalancer.ILoadBalancer;
    import com.netflix.loadbalancer.Server;
    import java.util.ArrayList;
    import java.util.List;
    import java.util.Random;
    
    /**
     * 自定义的随机策略
     * Created by CaoHui on 2020/4/5.
     */
    public class TulingRandomRule extends AbstractLoadBalancerRule {
        Random rand;
        public TulingRandomRule() {
            rand = new Random();
        }
        private int currentIndex = 0;
        private List<Server> currentChooseList = new ArrayList<Server>();
        /**
         * Randomly choose from all living servers
         */
        public Server choose(ILoadBalancer lb, Object key) {
            if (lb == null) {
                return null;
            }
            Server server = null;
            while (server == null) {
                if (Thread.interrupted()) {
                    return null;
                }
                List<Server> upList = lb.getReachableServers();
                List<Server> allList = lb.getAllServers();
    
                int serverCount = allList.size();
                if (serverCount == 0) {
                    return null;
                }
    
                //第一次进来 随机选取一个下标
                int index = rand.nextInt(serverCount);
    
                //当前轮询的次数小于等于5
                if(currentIndex<5) {
                    //保存当前选择的服务列表ip
                    if(currentChooseList.isEmpty()) {
                        currentChooseList.add(upList.get(index));
                    }
                    //当前的++
                    currentIndex++;
                    //返回保存的
                    return currentChooseList.get(0);
                }else {
                    currentChooseList.clear();
                    currentChooseList.add(0,upList.get(index));
                    currentIndex=0;
                }
    
                if (server == null) {
                    Thread.yield();
                    continue;
                }
    
                if (server.isAlive()) {
                    return (server);
                }
    
                server = null;
                Thread.yield();
            }
    
            return server;
        }
    
        @Override
        public Server choose(Object key) {
            return choose(getLoadBalancer(), key);
        }
    
        @Override
        public void initWithNiwsConfig(IClientConfig clientConfig) {
            // TODO Auto-generated method stub
    
        }
    }



