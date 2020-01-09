# training
## week1
### spring bean
  定义：组成应用程序的主体及由 Spring IOC 容器所管理的对象</br>
  
  生命周期概述：</br>
    1.bean对象的实例化</br>
    2.封装属性，也就是设置properties中的属性值</br>
    3.如果bean实现了BeanNameAware，则执行setBeanName方法,也就是bean中的id值</br>
    4.如果实现BeanFactoryAware或者ApplicationContextAware ，需要设置setBeanFactory或者上下文对象setApplicationContext</br>
    5.如果存在类实现BeanPostProcessor后处理bean，执行postProcessBeforeInitialization，可以在初始化之前执行一些方法</br>
    6.如果bean实现了InitializingBean，则执行afterPropertiesSet，执行属性设置之后的操作</br>
    7.调用<bean　init-method="">执行指定的初始化方法</br>
    8.如果存在类实现BeanPostProcessor则执行postProcessAfterInitialization，执行初始化之后的操作</br>
    9.执行自身的业务方法</br>
    10.如果bean实现了DisposableBean，则执行spring的的销毁方法</br>
    11.调用<bean　destory-method="">执行自定义的销毁方法。</br>
    
   各种接口方法分类：</br>
   1.Bean自身的方法: 这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法</br>
   2.Bean级生命周期接口方法: 这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法</br>
   3.容器级生命周期接口方法: 这个包括了InstantiationAwareBeanPostProcessor 和BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”</br>
   4.工厂后处理器接口方法: 这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。


