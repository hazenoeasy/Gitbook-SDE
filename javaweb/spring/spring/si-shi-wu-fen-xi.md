# 四：事务分析

## 1）@EnableTransactionManagement

利用TransactionManagementConfigurationSelector给容器中会导入组件

![1602684779186](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221300-687604.png)

![1602684860349](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221643-713143.png)

这个类就是继承了ImportSelect接口

![1602684842649](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221457-687682.png)

![1602684977103](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221651-80602.png)

这个adviceMode默认是PROXY

![1602685055068](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221736-11509.png)

![1602685107515](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221830-896637.png)

所以可以导入两个组件AutoProxyRegistrar和ProxyTransactionManagementConfiguration

![1602684842649](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014221457-687682.png)

## 2）AutoProxyRegistrar

实现了ImportBeanDefinitionRegistrar接口，给容器中注册一个 InfrastructureAdvisorAutoProxyCreator 组件

![1602685265190](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014222107-304564.png)

![1602685312816](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014222154-397805.png)

给容器中注册一个 InfrastructureAdvisorAutoProxyCreator 组件；

![1602685339308](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014222220-16469.png)

利用后置处理器机制在对象创建以后，包装对象，返回一个代理对象（增强器），代理对象执行方法利用拦截器链进行调用；

![1602685497408](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014222456-517188.png)

![1602685624273](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014222704-731665.png)

## 3）ProxyTransactionManagementConfiguration

1、给容器中注册**事务增强器**；

![1602687612869](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014230013-555172.png)

他这里需要一个事务属性transactionAttributeSource()

![1602687697901](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014230138-268583.png)

来看其中一个SpringTransactionAnnotationParser解析器

![1602687783977](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201016103854-390104.png)

**事务增强器**要用事务注解的信息（事务属性信息），解析事务注解，就是获取注解中各种属性的值

![1602687799800](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201016103900-415699.png)

回到事务增强器，这里还注册了一个事务拦截器：

![1602688017264](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201016103904-895222.png)

TransactionInterceptor 保存了事务属性信息transactionAttributeSource，事务管理器 txManager；

![1602688355649](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014231250-981585.png)

TransactionInterceptor其实还是一个方法拦截器

![1602688588216](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201016103909-981216.png)

看看TransactionInterceptor的invoke方法

![1602689038434](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201016103912-978241.png)

事务拦截器：

在目标方法执行的时候，执行拦截器链，只有一个拦截器，就是TransactionInterceptor；

1. 先获取事务相关的属性 transactionAttributeSource
2. 再获取PlatformTransactionManager，如果事先没有添加指定任何transactionmanger （在@Transactional注解中可以指定）；最终会从容器中按照类型获取一个PlatformTransactionManager；
3. 执行目标方法 如果异常，获取到事务管理器，利用事务管理回滚操作； 如果正常，利用事务管理器，提交事务

先获取事务相关的属性 transactionAttributeSource

![1602689081060](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014232442-298807.png)

再获取PlatformTransactionManager，如果事先没有添加指定任何transactionmanger 最终会从容器中按照类型获取一个PlatformTransactionManager；

![1602689497149](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014233138-592678.png)

回到上一步：

![1602690288089](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014234449-941382.png)

如果出现异常：

![1602690338198](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014234539-496260.png)

出现异常就拿到事务管理器进行回滚：

![1602690380275](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014234621-967522.png)

如果正常：

![1602690430883](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201016103920-64554.png)

拿到事务管理器进行提交：

![1602690450560](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20201014234731-434926.png)

整个过程是跟Aop的过程是一样的

```properties
1）、@EnableTransactionManagement
			利用TransactionManagementConfigurationSelector给容器中会导入组件
			导入两个组件
			AutoProxyRegistrar
			ProxyTransactionManagementConfiguration
2）、AutoProxyRegistrar：
			给容器中注册一个 InfrastructureAdvisorAutoProxyCreator 组件；
			InfrastructureAdvisorAutoProxyCreator：？
			利用后置处理器机制在对象创建以后，包装对象，返回一个代理对象（增强器），代理对象执行方法利用拦截器链进行调用；
3）、ProxyTransactionManagementConfiguration 做了什么？
			1、给容器中注册事务增强器；
				1）、事务增强器要用事务注解的信息，AnnotationTransactionAttributeSource解析事务注解
				2）、事务拦截器：
					TransactionInterceptor；保存了事务属性信息，事务管理器；
					他是一个 MethodInterceptor；
					在目标方法执行的时候；
						执行拦截器链；
						事务拦截器：
							1）、先获取事务相关的属性
							2）、再获取PlatformTransactionManager，如果事先没有添加指定任何transactionmanger
								最终会从容器中按照类型获取一个PlatformTransactionManager；
							3）、执行目标方法
								如果异常，获取到事务管理器，利用事务管理回滚操作；
								如果正常，利用事务管理器，提交事务
```
