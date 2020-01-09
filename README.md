# training
## week1
### spring bean
  定义：组成应用程序的主体及由 Spring IOC 容器所管理的对象
  生命周期：
    1.bean对象的实例化
    2.封装属性，也就是设置properties中的属性值
    3.如果bean实现了BeanNameAware，则执行setBeanName方法,也就是bean中的id值
    4.如果实现BeanFactoryAware或者ApplicationContextAware ，需要设置setBeanFactory或者上下文对象setApplicationContext
    5.如果存在类实现BeanPostProcessor后处理bean，执行postProcessBeforeInitialization，可以在初始化之前执行一些方法
    6.如果bean实现了InitializingBean，则执行afterPropertiesSet，执行属性设置之后的操作
    7.调用<bean　init-method="">执行指定的初始化方法
    8.如果存在类实现BeanPostProcessor则执行postProcessAfterInitialization，执行初始化之后的操作
    9.执行自身的业务方法
    10.如果bean实现了DisposableBean，则执行spring的的销毁方法
    11.调用<bean　destory-method="">执行自定义的销毁方法。
