##  Bean和BeanDefinition

#### Bean是Spring的一等公民

+ Bean的本质就是Java对象，只是这个对象的生命周期有容器来管理
+ 不需要为了创建Bean在原来的Java类上添加任何额外的限制
+ 对Java对象的控制体现在配置文件上



#### BeanDefinition : 即Bean的定义

根据配置，生成用来描述Bean的BeanDefinition，常用属性:

+ 作用范围scope(@Scope)   singleton-全局有且仅有一个    prototype--每次获取bean就会有一个新的实例
+ 懒加载lazy-init(@Lazy)  决定Bean实例是否延迟加载
+ 首选primary(@Primary) 设置为True的bean是会有限加载的
+ factory-bean和factory-method （@Configuration和 @Bean）



![image-20200714154304325](D:\MyStudy\学习杂记\Java\Spring\Bean和BeanDefinition.assets\image-20200714154304325.png)

1. 读取配置文件到内存中 ；配置文件可以是xml也可以是注解
2. 在内存中，这些配置信息会被当做一个个Resource对象
3. 然后解析成BeanDefinition实例
4. 最后注册到容器中 