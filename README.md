# training
## week1
### spring bean
  定义：组成应用程序的主体及由 Spring IOC 容器所管理的对象</br>
  
  生命周期概述：</br>
    1.Bean容器找到配置文件中 Spring Bean 的定义</br>
    2.Bean容器利用Java Reflection API创建一个Bean的实例</br>
    3.如果涉及到一些属性值 利用set方法设置一些属性值</br>
    4.如果Bean实现了BeanNameAware接口，调用setBeanName()方法，传入Bean的名字</br>
    5.如果Bean实现了BeanClassLoaderAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例</br>
    6.如果Bean实现了BeanFactoryAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。</br>
    7.与上面的类似，如果实现了其它Aware接口，就调用相应的方法。</br>
    8.如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessBeforeInitialization()方法。</br>
    9.如果Bean实现了InitializingBean接口，执行afterPropertiesSet()方法。</br>
    10.如果Bean在配置文件中的定义包含init-method属性，执行指定的方法。</br>
    11.如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象，执行postProcessAfterInitialization()方法</br>
    12.当要销毁Bean的时候，如果Bean实现了DisposableBean接口，执行destroy()方法。</br>
    13.当要销毁Bean的时候，如果Bean在配置文件中的定义包含destroy-method属性，执行指定的方法。</br>
  流程图如下：
    ![avatar](C:\Users\HCAO25\Desktop\bean.jpg)



    
   各种接口方法分类：</br>
   1.Bean自身的方法: 这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法</br>
   2.Bean级生命周期接口方法: 这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法</br>
   3.容器级生命周期接口方法: 这个包括了InstantiationAwareBeanPostProcessor 和BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”</br>
   4.工厂后处理器接口方法: 这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。


