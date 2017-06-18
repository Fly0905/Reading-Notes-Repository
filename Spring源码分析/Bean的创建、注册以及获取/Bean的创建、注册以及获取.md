[TOC]

# Bean的创建、注册以及获取

**前言：**

1.Spring基于XML配置bean和基于编程式(@Bean)配置bean的创建前解析工作有点不一样，本文将会分开讲解；

2.在展开bean的创建过程之前，先列举一下Spring的bean创建过程(其实不仅仅包括创建过程，还有注册、扩展等操作，下文简单称为bean创建过程)需要到的核心类。

## 准备阶段

先简单介绍一下Spring基于XML文件的bean创建过程中使用到的核心类，主要包括三个部分：

### 1.容器加载的相关类(BeanFactory体系)

![beanFactory_relations](/beanFactory_relations.png)

下面逐层介绍一下上图BeanFactory体系中涉及到的所有接口和类：

* **AliasRegistry: **定义对alias(**别名**)的简单增删查等操作。

* **SimpleAliasRegistry: ** 主要使用map(**ConcurrentHashMap**)作为alias的缓存，并且实现AliasRegistry接口。

* **SingletonBeanRegistry: **定义对单例的注册和获取。

* **BeanFactory: ** 接口定义了基本的IoC容器的规范，如获取bean和获取判断bean全局范围类型等。

* **DefaultSingletonBeanRegistry: ** 对SingletonBeanRegistry接口的实现，里面有多个集合类型的容器用于判断或者存储单例bean或者用于bean创建过程时的各种判断(常听说的Spring单例缓存池就在这里)。

* **HierarchicalBeanFactory: ** 它主要是提供父 BeanFactory 的功能，通过它能够获取当前 BeanFactory 的父工厂（ PS: 若在 A 工厂启动并加载 Bean 之前， B 工厂先启动并加载了，那 B 就是 A 的父工厂），这样就能让当前的 BeanFactory 加载父工厂加载的 Bean 了(和命名一致，使得BeanFactory得到层次化的功能)。

* **ListableBeanFactory: **提供了列举 Bean 的功能，他能够列举当前 BeanFactory 加载的所有 Bean ：列举所有 Bean 的名字或者满足某种类型的 Bean 的名字，根据类型返回所有 Bean 对象等。

* **BeanDefinitionRegistry: ** 定义对BeanDefinition的注册、移除、查询等功能。

* **FactoryBeanRegistrySupport: ** 在DefaultSingletonBeanRegistry基础上增加了对FactoryBean的特殊处理功能。

* **ConfigurableBeanFactory: ** 提供了配置BeanFactory的多种方法，例如添加BeanClassLoader、设置ParentBeanFactory、添加BeanPostProcessor等等。

* **AbstractBeanFactory: ** 综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能，有一点比较重要的是bean的大多数属性在这个BeanFactory里面缓存，例如PropertyEditor、BeanPostProcessor等。

* **AutowireCapableBeanFactory: ** 定义创建bean、bean自动注入、初始化bean(资源和属性)、应用bean的后处理器等方法。

* **AbstractAutowireCapableBeanFactory: ** 综合AbstractBeanFactory的功能并对AutowireCapableBeanFactory接口进行实现。

* **ConfigurableListableBeanFactory: **  提供了列举 Bean 的功能，添加指定忽略依赖、接口等。(继承自

  ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory三个接口，得到它们的所有功能)。

* **DefaultListableBeanFactory : ** 整个体系最底层实现，综合了上面所有功能，主要是对Bean注册后的处理。一般来说，在一些钩子接口拿到的BeanFactory的实例通常都是DefaultListableBeanFactory 的实例。

其实DefaultListableBeanFactory还有一个子类XmlBeanFactory，这个类是XML加载Bean的基础，但是不知道什么原因这个类在Spring 3.1版本已经废弃，而且注释里面写了废物的原因是"in favor of DefaultListableBeanFactory"(为了支持DefaultListableBeanFactory)，具体原因不得而知，但是它仍然是Spring

XML配置Bean的加载入口(DEBUG的时候断点可以放在**this.reader.loadBeanDefinitions(resource)**)。

### 2.XML配置文件读取操作相关类

![eanFactory_relations](/resource_reader.png)

简述一下XML配置文件读取的大概步骤：

(1) 入口类是XmlBeanDefinitionReader，通过继承于AbstractBeanDefinitionReader中的方法通过ResourceLoader中的location把配置文件转化为Resource(Spring-core包的Resource)

(2) 通过DocumentLoader接口的实现类对Resource文件进行转换，将Resource转化为Document

(3)通过接口BeanDefinitionDocumentReader的实现类DefaultBeanDefinitionDocumentReader对

Document进行解析，Element的解析实际上是由BeanDefinitionParserDelegate完成的。

### 3.资源文件的类族

![eanFactory_relations](/resource_class.png)

最主要是ClassPathResource，它的实现最底层依赖于class或者classLoader提供底层方法实现，用于加载类路径下的资源文件，作用是实现资源文件转换为InputStream。

### 4.BeanDefinition的类族

![eanFactory_relations](/beandefinition_class.png)

(1)BeanDefinition是bean定义的接口

(2)