[TOC]

# Bean的创建、注册以及获取

**前言：**

1.Spring基于XML配置bean和基于编程式(@Bean)配置bean的创建前解析工作有点不一样，本文只会讲解基于编程式配置Bean的情况；

2.在展开bean的创建过程之前，先列举一下Spring的bean创建过程(其实不仅仅包括创建过程，还有注册、扩展等操作，下文简单称为bean创建过程)需要到的核心类。

## 准备阶段

先简单介绍一下Spring基于XML文件的bean创建过程中使用到的核心类，主要包括三个部分：

### 1.容器加载的相关类(BeanFactory体系)

![beanFactory_relations](beanFactory_relations.png)

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

![eanFactory_relations](resource_reader.png)

简述一下XML配置文件读取的大概步骤：

(1) 入口类是XmlBeanDefinitionReader，通过继承于AbstractBeanDefinitionReader中的方法通过ResourceLoader中的location把配置文件转化为Resource(Spring-core包的Resource)

(2) 通过DocumentLoader接口的实现类对Resource文件进行转换，将Resource转化为Document

(3)通过接口BeanDefinitionDocumentReader的实现类DefaultBeanDefinitionDocumentReader对

Document进行解析，Element的解析实际上是由BeanDefinitionParserDelegate完成的。

### 3.资源文件的类族

![eanFactory_relations](resource_class.png)

最主要是ClassPathResource，它的实现最底层依赖于class或者classLoader提供底层方法实现，用于加载类路径下的资源文件，作用是实现资源文件转换为InputStream。

### 4.BeanDefinition的类族

![eanFactory_relations](beandefinition_class.png)

(1)BeanDefinition是bean定义的接口，定义了一些接口规范。

(2)AbstractBeanDefinition是bean定义的抽象类，主要是存放BeanDefinition的默认属性和公有属性，并且重写了重写了equals，hashCode，toString方法。

(3)RootBeanDefinition和ChildBeanDefinition，一个RootBeanDefinition定义表明它是一个可合并的bean definition，表示一个从配置源（XML，Java Config）中生成的BeanDefinition，ChildBeanDefinition是一种bean definition，它可以继承它父类的设置，即ChildBeanDefinition对RootBeanDefinition有一定的依赖关系，ChildBeanDefinition从父类RootBeanDefinition继承构造参数值，属性值并可以重写父类的方法，同时也可以增加新的属性或者方法。(类同于java类的继承关系)。若指定初始化方法，销毁方法或者静态工厂方法，　　ChildBeanDefinition将重写相应父类的设置，depends on，autowire mode，dependency check，sigleton，lazy init 一般由子类自行设定。

(4)GenericBeanDefinition是一站式的标准bean definition，除了具有指定类、可选的构造参数值和属性参数这些其它bean definition一样的特性外，它还具有通过parenetName属性来灵活设置parent bean definition，这个类在Spring2.5后可以直接取代RootBeanDefinition和ChildBeanDefinition的组合。

(5)AnnotatedGenericBeanDefinition，对应于**注解@Bean上的目标方法返回的目标对象或者内建注解@Component对应的目标类生成的目标对象**，继承于GenericBeanDefinition，实现了AnnotatedBeanDefinition接口，具备获取目标对象注解元信息的功能。

下面简单叙述一下BeanDefinition体系里面的一些接口和抽象类提供的重要方法：

AnnotatedBeanDefinition：提供获取注解元信息和方法元信息。

```java
public interface AnnotatedBeanDefinition extends BeanDefinition {

	/**
	 * Obtain the annotation metadata (as well as basic class metadata)
	 * for this bean definition's bean class.
	 * @return the annotation metadata object (never {@code null})
	 */
	AnnotationMetadata getMetadata();

	/**
	 * Obtain metadata for this bean definition's factory method, if any.
	 * @return the factory method metadata, or {@code null} if none
	 * @since 4.1.1
	 */
	MethodMetadata getFactoryMethodMetadata();

}
```

AttributeAccessor：定义了最基本的对成员属性的增删查改。

```java
public interface AttributeAccessor {

	void setAttribute(String name, Object value);

	Object getAttribute(String name);

	Object removeAttribute(String name);
	
	boolean hasAttribute(String name);

	String[] attributeNames();
}
```

BeanMetadataElement：定义了一个可以返回配置源的元信息的方法。

```java
public interface BeanMetadataElement {

	/**
	 * Return the configuration source {@code Object} for this metadata element
	 * (may be {@code null}).
	 */
	Object getSource();

}
```

抽象类AttributeAccessorSupport：实现了AttributeAccessor，把成员属性存放在一个叫attributes的LinkedHashMap中。

抽象类BeanMetadataAttributeAccessor：继承于AttributeAccessorSupport，作用有点像一个适配器，原来的添加成员属性是设置key-value，在这里适配为#addMetadataAttribute(BeanMetadataAttribute attribute)，同时提供了公有方法#setSource(Object source)。

BeanDefinitionHolder实现了BeanMetadataElement，同时，它的属性如下：

```java
    private final BeanDefinition beanDefinition;

	private final String beanName;

	private final String[] aliases;
```

这个类持有BeanDefinition对象，beanName以及bean的别名，在Spring内部，它用来临时保存BeanDefinition来传递BeanDefinition，作用就如命名那样，BeanDefinition的持有者。



## 刷新容器上下文(refreshContext)

Spring启动的一个最核心的操作就是刷新容器上下文，这个操作包括很多步骤，下面一一详细分析。

入口是SpringApplication里面的一个#refreshContext方法：

```java
private void refreshContext(ConfigurableApplicationContext context) {
		refresh(context);
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}
```

调用链的末端委托为AbstractApplicationContext的#refresh方法：

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

这个方法里面有十多个小的方法，看似简单，但是每个小方法里面都有一套极长的调用链，下面分小节详细讲解每个小方法的内容。

### prepareRefresh

AbstractApplicationContext#prepareRefresh：刷新上下文前的准备方法，主要是初始化资源文件的源和校验是否缺少必须的资源属性。

```java
    protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
```

* 方法前面三行分别是记录启动时间戳，把closed设置为false，把active设置为true。

* initPropertySources：初始化资源属性源，主要是构造ConfigurableEnvironment的实例StandardEnvironment，同时如果构造的ConfigurableEnvironment是ConfigurableWebEnvironment的实例，将会使用servletContext初始化资源属性源，详见StandardServletEnvironment#initPropertySources。

* validateRequiredProperties：校验必须的资源属性值，此方法最后委托给AbstractPropertyResolver#

  validateRequiredProperties，主要做requiredProperties的非空校验。

* 构建一个名为earlyApplicationEvents，元素类型为ApplicationEvent的LinkedHashSet，作用见它的注释描述：构造一个早期的ApplicationEvent集合用于**multicaster**(广播器)被激活时ApplicationEvent发布。

### obtainFreshBeanFactory

AbstractApplicationContext#obtainFreshBeanFactory（**这个方法十分重要**），作用是获取新的BeanFactory。

```java
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```

首先我们一定要关注一个十分主要的类**GenericApplicationContext**，这个类的空入参构造就是生成一个新的**DefaultListableBeanFactory**，熟悉Spring体系的人必然知道DefaultListableBeanFactory这个类的重要性，obtainFreshBeanFactory方法如下，里面的#refreshBeanFactory和#getBeanFactory都是由GenericApplicationContext实现，refreshBeanFactory主要是把GenericApplicationContext中的#refreshBeanFactory通过cas设置为true，并且为DefaultListableBeanFactory设置serializationId，#getBeanFactory就是返回新建的DefaultListableBeanFactory实例。

那么这里有一个比较大的疑惑，GenericApplicationContext的实例是什么时候创建的？答案见SpringBootServletInitializer的#createRootApplicationContext：

```java
   protected WebApplicationContext createRootApplicationContext(
			ServletContext servletContext) {
		SpringApplicationBuilder builder = createSpringApplicationBuilder();
		builder.main(getClass());
		ApplicationContext parent = getExistingRootWebApplicationContext(servletContext);
		if (parent != null) {
			this.logger.info("Root context already created (using as parent).");
			servletContext.setAttribute(
					WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, null);
			builder.initializers(new ParentContextApplicationContextInitializer(parent));
		}
		builder.initializers(
				new ServletContextApplicationContextInitializer(servletContext));
		builder.listeners(new ServletContextApplicationListener(servletContext));
		builder.contextClass(AnnotationConfigEmbeddedWebApplicationContext.class);
		builder = configure(builder);
		SpringApplication application = builder.build();
		if (application.getSources().isEmpty() && AnnotationUtils
				.findAnnotation(getClass(), Configuration.class) != null) {
			application.getSources().add(getClass());
		}
		Assert.state(!application.getSources().isEmpty(),
				"No SpringApplication sources have been defined. Either override the "
						+ "configure method or add an @Configuration annotation");
		// Ensure error pages are registered
		if (this.registerErrorPageFilter) {
			application.getSources().add(ErrorPageFilterConfiguration.class);
		}
		return run(application);
	}
```

builder.contextClass(AnnotationConfigEmbeddedWebApplicationContext.class);这一行代码就是设置了当前应用的上下文的class，**AnnotationConfigEmbeddedWebApplicationContext**继承于EmbeddedWebApplicationContext，而EmbeddedWebApplicationContext继承于GenericApplicationContext，同时它具备了AnnotationConfigWebApplicationContext的功能，这个被强化的上下文类具备注解配置、嵌入式初始化启动、Web应用上下文等功能。这里仅仅是设置了context的class，真正被实例化在SpringApplication的#run( String... args)方法里面的**context = createApplicationContext( )**，九曲十八弯后终于弄清楚获取新的BeanFactory的详细过程。

### prepareBeanFactory

对上一步obtainFreshBeanFactory创建的BeanFactory进行一些准备操作。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

主要做了下面几件事：

* 设置BeanClassLoader、BeanExpressionResolver(SPEL解析相关)，PropertyEditorRegistrar(PropertyEditor注册器)。
* 添加ApplicationContextAwareProcessor同时忽略所有Aware族接口。
* 注册几个可解析的依赖，包括BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext，其实后面三者都是AnnotationConfigEmbeddedWebApplicationContext的实例。
* 添加一个BeanPostProcessor为ApplicationListenerDetector(ApplicationListener检测器)。
* 添加一个BeanPostProcessor为LoadTimeWeaverAwareProcessor(用于织入第三方模块，如AspectJ，目标类需要实现LoadTimeWeaverAware接口)，为BeanFactory添加一个ContextTypeMatchClassLoader，同时为ContextTypeMatchClassLoader添加ClassLoader。
* 剩下就是注册几个必须的单例，分别是ConfigurableEnvironment实例，SystemProperties实例(本质上是Map)和SystemEnvironment(本质上是Map)。

### postProcessBeanFactory

AbstractApplicationContext#postProcessBeanFactory、AbstractApplicationContext#invokeBeanFactoryPostProcessors、AbstractApplicationContextregisterBeanPostProcessors这三个方法是刷新Spring应用上下文的重要方法。AbstractApplicationContext#postProcessBeanFactory方法的详细内容见GenericWebApplicationContext#postProcessBeanFactory，依赖于前边创建的ConfigurableListableBeanFactory实例。

```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);

		WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext);
	}
```

分析下大致过程：

* 添加一个BeanPostProcessor为ServletContextAwareProcessor，同时BeanFactory添加一个忽略的接口ServletContextAware。
* 方法registerWebApplicationScopes：注册和Web应用Scope相关的对象。

```java
public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory, ServletContext sc) {
		beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope());
		beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope(false));
		beanFactory.registerScope(WebApplicationContext.SCOPE_GLOBAL_SESSION, new SessionScope(true));
		if (sc != null) {
			ServletContextScope appScope = new ServletContextScope(sc);
			beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope);
			// Register as ServletContext attribute, for ContextCleanupListener to detect it.
			sc.setAttribute(ServletContextScope.class.getName(), appScope);
		}

		beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
		beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
		beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
		beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
		if (jsfPresent) {
			FacesDependencyRegistrar.registerFacesDependencies(beanFactory);
		}
	}
```

beanFactory#registerScope把Scope的实例注册并且存放在AbstractBeanFactory的一个属性名为scopes的LinkedHashMap里面，同时通过beanFactory#registerResolvableDependency注册几个基于ObjectFactory实例获取的接口的实例。#registerResolvableDependency的作用是注册那些依赖ObjectFactory 注入接口实例的对象，如果有疑问可以看下上面的几个类RequestObjectFactory、ResponseObjectFactory等等的用法。

* registerEnvironmentBeans方法主要是注册和Web应用上下文环境相关的单例Bean:

```java
public static void registerEnvironmentBeans(
			ConfigurableListableBeanFactory bf, ServletContext servletContext, ServletConfig servletConfig) {

		if (servletContext != null && !bf.containsBean(WebApplicationContext.SERVLET_CONTEXT_BEAN_NAME)) {
			bf.registerSingleton(WebApplicationContext.SERVLET_CONTEXT_BEAN_NAME, servletContext);
		}

		if (servletConfig != null && !bf.containsBean(ConfigurableWebApplicationContext.SERVLET_CONFIG_BEAN_NAME)) {
			bf.registerSingleton(ConfigurableWebApplicationContext.SERVLET_CONFIG_BEAN_NAME, servletConfig);
		}

		if (!bf.containsBean(WebApplicationContext.CONTEXT_PARAMETERS_BEAN_NAME)) {
			Map<String, String> parameterMap = new HashMap<String, String>();
			if (servletContext != null) {
				Enumeration<?> paramNameEnum = servletContext.getInitParameterNames();
				while (paramNameEnum.hasMoreElements()) {
					String paramName = (String) paramNameEnum.nextElement();
					parameterMap.put(paramName, servletContext.getInitParameter(paramName));
				}
			}
			if (servletConfig != null) {
				Enumeration<?> paramNameEnum = servletConfig.getInitParameterNames();
				while (paramNameEnum.hasMoreElements()) {
					String paramName = (String) paramNameEnum.nextElement();
					parameterMap.put(paramName, servletConfig.getInitParameter(paramName));
				}
			}
			bf.registerSingleton(WebApplicationContext.CONTEXT_PARAMETERS_BEAN_NAME,
					Collections.unmodifiableMap(parameterMap));
		}

		if (!bf.containsBean(WebApplicationContext.CONTEXT_ATTRIBUTES_BEAN_NAME)) {
			Map<String, Object> attributeMap = new HashMap<String, Object>();
			if (servletContext != null) {
				Enumeration<?> attrNameEnum = servletContext.getAttributeNames();
				while (attrNameEnum.hasMoreElements()) {
					String attrName = (String) attrNameEnum.nextElement();
					attributeMap.put(attrName, servletContext.getAttribute(attrName));
				}
			}
			bf.registerSingleton(WebApplicationContext.CONTEXT_ATTRIBUTES_BEAN_NAME,
					Collections.unmodifiableMap(attributeMap));
		}
	}
```

这几个Bean的命名存放在WebApplicationContext接口的常量，实际上这些对象大多是集合类对象，注册操作有点类似Environment的实例注册。

### invokeBeanFactoryPostProcessors

AbstractApplicationContext#invokeBeanFactoryPostProcessors，实现和处理内建的BeanFactoryPostProcessor接口的实例。一般，我们要获取指定的ClassPath下的所有类，必须要扫描指定包下面的所有类，而扫描指定包下的所有类这个动作是由AbstractApplicationContext#invokeBeanFactoryPostProcessors的深层调用链触发的，因此这个步骤在整个上下文刷新的操作中是十分重要的。

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
}
```

核心处理逻辑委托给PostProcessorRegistrationDelegate，这个类作用就是PostProcessor的注册，是一个委托(实际上Java里面不存在委托，委托也并不是设计模式，Spring里面使用的是"委托"的语义，而且多处用到)工具类。这个类主要处理两类PostProcessor接口的实例：BeanFactoryPostProcessor和BeanDefinitionRegistryPostProcessor。首先要了解BeanDefinitionRegistryPostProcessor这个接口的实例可以实现传递形式的注册，举个例子，A、B同时实现了BeanDefinitionRegistryPostProcessor，A被激活，执行BeanDefinitionRegistryPostProcessor的目标方法的时候可以把B进行注册，然后一直传递下去。





未完待续.....

