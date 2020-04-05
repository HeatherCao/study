---
layout: post
title: springcloud组件——Feign
keywords: Feign
description: Feign
categories: [springCloud, Feign]
tags: [springCloud, Feign]
---

*  目录
{:toc}

# Feign
## 什么是Feign

    Feign是Netflix开发的声明式、模板化的HTTP客户端，其灵感来自Retrofit、JAXRS-2.0以及WebSocket。Feign可帮助我们更加便捷、优雅地调用HTTP API。
    在Spring Cloud中，使用Feign非常简单——只需创建接口，并在接口上添加注解即可。
    Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。Spring Cloud对Feign进行了增强，使其支持Spring MVC注解，另外还整合了Ribbon和Eureka，从而使得Feign的使用更加方便
    
    eg: 以前我们调用dao的时候 ，我们是不是一个接口加 注解然后在service中就可以进行调用了
    @Mapper
    public interface OrderDao {
        List<Order> queryOrdersByUserId(Integer userId);
    }
    
## Feign VS Ribbon

    Ribbon+RestTemplate进行微服务调用
    缺点：
    ResponseEntity<List> responseEntity = restTemplate.getForEntity("http://order-service/order/queryOrdersByUserId/"+userId,List.class);
    
    1.我们不难发现，我们构建上诉的URL 是比较简单的，假如我们业务系统十分复杂，类似如下节点
    https://www.baidu.com/s?wd=asf&rsv_spt=1&rsv_iqid=0xa25bbeba000047fd&issp=1&f=8&rsv_bp=0&rsv_idx=2&ie=utf-8&tn=baiduhome_pg&rsv_enter=1&rsv_sug3=3&rsv_sug1=2&rsv_sug7=100&rsv_sug2=0&inputT=328&rsv_sug4=328
    
    那么我们构建这个请求的URL是不是很复杂，若我们请求的参数有可能变动，那么是否这个URL是不是很复杂
    2.如果系统业务非常复杂，而你是一个新人，当你看到这行代码，恐怕很难一眼看出其用途是什么！此时，你很可能需要寻求老同事的帮助（往往是这行代码的作者，哈哈哈，可万一离职了呢？），或者查阅该目标地址对应的文档（文档常常还和代码不匹配），才能清晰了解这行代码背后的含义！否则，你只能陷入蛋疼的境地！
     
## Feign的使用

    正接口上的name指定微服务的名称,  path是指定服务消费者的调用前缀
    @FeignClient(name = "ms-provider-order",path = "/order")
    public interface OrderApi {
    
        @RequestMapping("/queryOrdersByUserId/{userId}")
        public List<OrderVo> queryOrdersByUserId(@PathVariable("userId") Integer userId);
    }
    
    消费端主启动类上加入注解@EnableFeignClients(basePackages = "com.tuling")
    
## 如何自定义Feign
### 自定义日志级别
#### 日志级别

    NONE【性能最佳，适用于生产】：不记录任何日志（默认值）。
    HEADERS：记录BASIC级别的基础上，记录请求和响应的header。
    FULL【比较适用于开发及测试环境定位问题】：记录请求和响应的header、body和元数据。
    BASIC【适用于生产环境追踪问题】：仅记录请求方法、URL、响应状态代码以及执行时间。

#### 代码中如何自定义

    配置类
    public class MsProvider8007CustomCfg {
        @Bean
        public Logger.Level feignLoggerLevel() {
            return Logger.Level.BASIC;
        }
    
    }
    
    @FeignClient(name = "ms-provider-order-feign-custom01",configuration = MsProvider8007CustomCfg.class,path = "/order")
    public interface MsCustomFeignOrderApi {
    
        @RequestLine("GET /queryOrdersByUserId/{userId}")
        public List<OrderVo> queryOrdersByUserId(@Param("userId") Integer userId);
    }

#### yml中如何配置
    
    feign.client.config.ms-provider-order-feign-custom01.loggerLevel=full

### 如何支持feign的注解来替换springmvc的注解
    代码设置
    public class MsProvider8007CustomCfg {
     
        修改协议契约
        @Bean
        public Contract feignContract() {
            return new Contract.Default();
        }
    }
    
    @FeignClient(name = "ms-provider-order-feign-custom01",configuration = MsProvider8007CustomCfg.class,path = "/order")
    public interface MsCustomFeignOrderApi {
     @RequestLine("GET /queryOrdersByUserId/{userId}")
        public List<OrderVo> queryOrdersByUserId(@Param("userId") Integer userId);
    }
    
    配置文件设置
    feign.client.config.ms-provider-order-feign-custom01.contract=feign.Contract.Default
    
### 创建拦截器设置公用参数实现:RequestInterceptor
#### 编写拦截器

    public class TulingInterceptor implements RequestInterceptor {
        @Override
        public void apply(RequestTemplate template) {
            System.out.println("自定义拦截器");
            template.header("token","123456");
        }
    }
#### 加入拦截器配置

    代码配置
    public class MsProvider8007CustomCfg {
        @Bean
        public RequestInterceptor tulingInterceptor() {
            return new TulingInterceptor();
        }
    }
    
    配置文件配置
    feign.client.config.ms-provider-order-feign-custom01.requestInterceptors[0]=com.tuling.interceptor.TulingInterceptor
#### 服务提供端获取公共参数

    截图:
![](/images/2020-04-06-springcloud-feign-instructions-01.png)


