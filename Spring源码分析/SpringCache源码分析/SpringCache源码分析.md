[TOC]

# SpringCache源码分析

## 前言

如果对Spring Cache的使用尚不熟悉，可以参考一下[开涛的博客-Spring Cache抽象详解](http://jinnianshilongnian.iteye.com/blog/2001040)，这篇文章主要分析一下Spring Cache的源码，这部分源码主要在Spring-context和Spring-context-support这两个包下（本文主要分析Spring-context包下的cache包的代码，Spring-context-support包里面的cache包下的逻辑大致相同），另外，如果需要扩展有可能会用到aspects，有可能需要使用spring-aspects和spring-instrument。本文使用的Spring版本为4.3.8.RELEASE，对应的Springboot版本为1.5.3.RELEASE。



## 注意

* 缓存的本质是key-value。
* Spring-Aop的限制：（1）基于原生jdk代理，**自调用无效**；（2）基于字节码改造（Cglib），目标类必须是非final域，修饰符必须非private，因为是通过生成子类（目标类必须可以继承）实现。

Spring-Aop的第一点限制，其实官方文档也有详细说明，大致如下：

![spring-aop](/spring-aop.png)

而SpringCache中的所有注解的处理逻辑都是通过Aop实现，因此如果使用过程中出现不生效等情况，很有可能是因为使用了类内方法的自调用。



## Spring-context中cache的package结构

![spring-context](/spring-context.png)

* annotation包：存放@Cacheable、@CacheEvict等注解以及它们的配置和解析类。
* concurrent包：存放基于ConcurrentMap的Cache接口实现，对应的FactoryBean以及CacheManager接口的实现。
* config包：存放基于xml配置的cache的解析器和命名空间处理器。
* interceptor包：存放实现cache的aop相关的类。
* support包：主要存放CacheManager接口的实现类。
* Cache和CacheManager接口。



