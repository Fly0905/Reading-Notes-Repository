[TOC]

# Spring(boot)扩展类、注解、组件使用详解

## 前言

​        SpringFramework其实具有很高的扩展性，只是很少人喜欢挖掘那些扩展点，而且官方的Refrence也很少提到那些Hook类或Hook接口，至于是不是Spring官方有意为之就不得而知。

​       本文记录一些笔者目前了解的Spring的一些对外开放的扩展点、Hook类、注解和核心组件等等。如果发现文中有什么错误，希望多多交流指正，一切以Spring源码为准。文章编写基于Spring的版本为4.3.8.RELEASE，对应Springboot的版本为1.5.3.RELEASE。

​        本文使用软件Typora编写，如果使用其他markdown软件可能导致格式布局变形。



## Spring核心组件扩展点

### 1、Aware接口族

Spring中提供了各种Aware接口，方便从上下文中获取当前的运行环境，比较常见的几个子接口有：

BeanFactoryAware,BeanNameAware,ApplicationContextAware,EnvironmentAware，BeanClassLoaderAware等，这些Aware的作用都可以从命名得知，并且其使用也是十分简单。

例如我们经常看到SpringContext工具类：

```java
@Component
public final class SpringContextAssisor implements ApplicationContextAware {

	private static ApplicationContext applicationContext;

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		SpringContextAssisor.applicationContext = applicationContext;
	}

	public static Object getBeanDefinition(String name) {
		return applicationContext.getBean(name);
	}

	public static <T> T getBeanDefinition(String name, Class<T> clazz) {
		return applicationContext.getBean(name, clazz);
	}

}
```

实现ApplicationContextAware接口可以获取ApplicationContext

又例如想获取到当前的一个Spring Bean的BeanFactory：

```java
@Component
public class OneBean implements BeanFactoryAware {
    private BeanFactory beanFactory;
 
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}
```

一般来说，拿到的应该是DefaultListableBeanFactory，因为这个BeanFactory是BeanFactory一族的最底层的BeanFactory实现类，拥有所有父BeanFactory的功能。

其他的Aware可以自己尝试下功能。

**PS：Aware是可以自定义扩展的，具体可以参考下面这篇的博客**：

http://www.cnblogs.com/RunForLove/p/5828916.html

### 2、InitializingBean接口和DisposableBean接口

**InitializingBean**接口只有一个方法#afterPropertiesSet，作用是：当一个Bean实现InitializingBean，#afterPropertiesSet方法里面可以添加自定义的初始化方法或者做一些资源初始化操作(Invoked by a BeanFactory after it has set all bean properties supplied ==> "当BeanFactory 设置完所有的Bean属性之后才会调用#afterPropertiesSet方法")。

**DisposableBean**接口只有一个方法#destroy，作用是：当一个单例Bean实现DisposableBean，#destroy可以添加自定义的一些销毁方法或者资源释放操作(Invoked by a BeanFactory on destruction of a singleton ==>"单例销毁时由BeanFactory调用#destroy")

使用例子:

```java
@Component
public class ConcreteBean implements InitializingBean,DisposableBean {

	@Override
	public void destroy() throws Exception {
		System.out.println("释放资源");
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("初始化资源");
	}
}
```

### 3、ImportBeanDefinitionRegistrar接口

**功能：**

先看官方的注释

```java
/**
 * Interface to be implemented by types that register additional bean definitions when
 * processing @{@link Configuration} classes. Useful when operating at the bean definition
 * level (as opposed to {@code @Bean} method/instance level) is desired or necessary.
 *
 * <p>Along with {@code @Configuration} and {@link ImportSelector}, classes of this type
 * may be provided to the @{@link Import} annotation (or may also be returned from an
 * {@code ImportSelector}).
 *
 * <p>An {@link ImportBeanDefinitionRegistrar} may implement any of the following
 * {@link org.springframework.beans.factory.Aware Aware} interfaces, and their respective
 * methods will be called prior to {@link #registerBeanDefinitions}:
 * <ul>
 * <li>{@link org.springframework.context.EnvironmentAware EnvironmentAware}</li>
 * <li>{@link org.springframework.beans.factory.BeanFactoryAware BeanFactoryAware}
 * <li>{@link org.springframework.beans.factory.BeanClassLoaderAware BeanClassLoaderAware}
 * <li>{@link org.springframework.context.ResourceLoaderAware ResourceLoaderAware}
 * </ul>
 *
 * <p>See implementations and associated unit tests for usage examples.
```

翻译一下大概如下：

1.当处理Java编程式配置类(使用了@Configuration的类)的时候，ImportBeanDefinitionRegistrar接口的实现类可以注册额外的bean definitions;

2.ImportBeanDefinitionRegistrar接口的实现类必须提供给@Import注解或者是ImportSelector接口返回值

3.ImportBeanDefinitionRegistrar接口的实现类可能还会实现下面org.springframework.beans.factory.Aware接口中的一个或者多个，它们各自的方法优先于ImportBeanDefinitionRegistrar#registerBeanDefinitions被调用

org.springframework.beans.factory.Aware的部分接口如下：

* org.springframework.context.EnvironmentAware(读取或者修改Environment的变量)
 * org.springframework.beans.factory.BeanFactoryAware (获取Bean自身的Bean工厂)
 * org.springframework.beans.factory.BeanClassLoaderAware(获取Bean自身的类加载器)
 * org.springframework.context.ResourceLoaderAware(获取Bean自身的资源加载器)

个人理解：

1.首先需要自定义一个类去实现ImportBeanDefinitionRegistrar接口， #registerBeanDefinitions方法的参数有(使用了@Import的类型)元注解AnnotationMetadata以及BeanDefinitionRegistry(Bean注册相关方法的提供接口)，通过BeanDefinitionRegistry的方法可以实现BeanDefinition注册、移除等相关操作；

2.为了保证1生效，必须定义一个Java配置类(带有注解@Configuration)通过@Import指定1中定义的实现类

(但是实际上发现任何一个注解带有@Import只要使用在任意Java类就能生效，不一定需要该类带有@Configuration)

一个例子：

目标是通过自定义注解@EnableThrowable里面的targets属性指定需要注册进去Spring容器的class，当注解使用在@Configuration的类上，实现指定class的注册，然后可以使用@Autowire实现自动注入。

定义ImportBeanDefinitionRegistrar的实现类EnableThrowableRegistrar：

```java
public class EnableThrowableRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

	@Override
	public void setEnvironment(Environment environment) {
		System.out.println("JAVA_HOME:" + environment.getProperty("JAVA_HOME"));
	}

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
										BeanDefinitionRegistry registry) {
		Map<String, Object> annotationAttributes
				= importingClassMetadata.getAnnotationAttributes(EnableThrowable.class.getCanonicalName());
		Class<?>[] targets = (Class<?>[]) annotationAttributes.get("targets");
		if (null != targets && targets.length > 0) {
			for (Class<?> target : targets) {
				BeanDefinition beanDefinition = BeanDefinitionBuilder
						.genericBeanDefinition(target)
						.getBeanDefinition();
				registry.registerBeanDefinition(beanDefinition.getBeanClassName(),
						beanDefinition);
			}
		}
	}
}
```

定义一个注解@EnableThrowable：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(value = {EnableThrowableRegistrar.class})
public @interface EnableThrowable {

	Class<?>[] targets() default {};
}
```

定义一个Java配置类ConcreteConfiguration：

```java
@Configuration
@EnableThrowable(targets = {ConcreteService.class})
public class ConcreteConfiguration {

}
```

定义一个非Spring管理的Service类ConcreteService：

```java
public class ConcreteService {

	public void sayHello(){
		System.out.println("ConcreteService say hello!");
	}
}
```

测试代码：

```java
@SpringBootTest(classes = Application.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class ConcreteServiceTest {

	@Autowired
	private ConcreteService concreteService;

	@Test
	public void sayHello() throws Exception {
		concreteService.sayHello();
	}

}
```

结果：

控制台输出：

```
JAVA_HOME:D:\jdk1.8
ConcreteService say hello!
```

可以看到读取Environment属性成功，同时普通Java类ConcreteService成功注册到Spring容器并且自动注入和调用成功。

### 4、BeanPostProcessor接口和BeanFactoryPostProcessor接口

一般我们叫这两个接口为**Spring的Bean后置处理器接口**,作用是为Bean的初始化前后提供可扩展的空间。先看接口的方法：

**BeanPostProcessor**

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
  
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

**BeanFactoryPostProcessor**

```java
public interface BeanFactoryPostProcessor {
 void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;  
}
```

BeanFactoryPostProcessor可以对bean的定义（**配置元数据**）进行处理。也就是说，Spring IoC容器允许BeanFactoryPostProcessor在容器实际实例化任何其它的bean之前读取配置元数据，并有可能修改它。如果你愿意，你可以配置多个BeanFactoryPostProcessor。你还能通过设置'order'属性来控制BeanFactoryPostProcessor的执行次序。(大概可以这样理解:**Spring容器加载了bean的定义文件之后，在bean实例化之前执行的**)

实现BeanPostProcessor接口可以在Bean(**实例化之后**)**初始化的前后**做一些自定义的操作，但是拿到的参数只有BeanDefinition实例和BeanDefinition的名称，也就是无法修改BeanDefinition元数据,这里说的Bean的初始化是：

1）bean实现了InitializingBean接口，对应的方法为afterPropertiesSet

2）在bean定义的时候，通过init-method设置的方法

**PS:BeanFactoryPostProcessor回调会先于BeanPostProcessor**

使用例子：

实现一个BeanPostProcessor==>ConcreteBeanPostProcessor

```java
@Order(1)
@Component
public class ConcreteBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (beanName.contains("postBean"))
		System.out.println(String.format("Bean初始化之前,bean:%s,beanName:%s", bean.toString(), beanName));
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (beanName.contains("postBean"))
		System.out.println(String.format("Bean初始化之后,bean:%s,beanName:%s", bean.toString(), beanName));
		return bean;
	}
}
```

实现一个BeanFactoryPostProcessor==>ConcreteBeanFactoryPostProcessor

```java
@Component
public class ConcreteBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		BeanDefinition beanDefinition = beanFactory.getBeanDefinition("postBean");
		MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();
		propertyValues.addPropertyValue("author", "throwable");
	}
}
```

定义一个Spring的Bean

```java
@Component
public class PostBean {

	private String author;

	public String getAuthor() {
		return author;
	}

	public void setAuthor(String author) {
		this.author = author;
	}

	public void sayhello() {
		System.out.println(String.format("author %s say hello!", author));
	}
}
```

测试类：

```java
@SpringBootTest(classes = Application.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class PostBeanTest {
	@Autowired
	private PostBean postBean;
	
	@Test
	public void sayhello() throws Exception {
		postBean.sayhello();
	}
}
```

结果：

控制台输出：

```
Bean初始化之前,bean:org.throwable.processor.PostBean@xxxxx,beanName:postBean
Bean初始化之后,bean:org.throwable.processor.PostBean@xxxxx,beanName:postBean
author throwable say hello!
```

**PS:有兴趣可以看下Spring内置的一些实现了后置处理器接口的方法，大概有下面这些：**

AnnotationAwareAspectJAutoProxyCreator  
AspectJAwareAdvisorAutoProxyCreator  
InitDestroyAnnotationBeanPostProcessor 
ApplicationContextAwareProcessor 
AutowiredAnnotationBeanPostProcessor 
CommonAnnotationBeanPostProcessor 
RequiredAnnotationBeanPostProcessor 
PersistenceAnnotationBeanPostProcessor 

...

### 5、BeanDefinitionRegistryPostProcessor 接口

BeanDefinitionRegistryPostProcessor 接口可以看作是BeanFactoryPostProcessor和ImportBeanDefinitionRegistrar的功能集合，既可以获取和修改BeanDefinition的元数据，也可以实现BeanDefinition的注册、移除等操作。

例子：

定义一个BeanDefinitionRegistryPostProcessor==>ConcreteBeanDefinitionRegistryPostProcessor

```java
@Component
public class ConcreteBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

	private static final String beanName = "concreteRPBean";

	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		BeanDefinition beanDefinition = BeanDefinitionBuilder
				.genericBeanDefinition(ConcreteRPBean.class)
				.getBeanDefinition();
		registry.registerBeanDefinition(beanName, beanDefinition);
	}

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
		MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();
		propertyValues.addPropertyValue("author", "throwable");
	}
}
```

定义一个普通的Java类：

```java
public class ConcreteRPBean {

	private String author;

	public String getAuthor() {
		return author;
	}

	public void setAuthor(String author) {
		this.author = author;
	}

	public void sayHello(){
		System.out.println(String.format("ConcreteRPBean call sayhello method ==> author %s say hello!", author));
	}
}
```

测试类：

```JAVA
@SpringBootTest(classes = Application.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class ConcreteRPBeanTest {

	@Autowired
	private ConcreteRPBean concreteRPBean;

	@Test
	public void sayHello() throws Exception {
		concreteRPBean.sayHello();
	}

}
```

结果：

控制台输出：

```
ConcreteRPBean call sayhello method ==> author throwable say hello!
```

### 6、FactoryBean接口

首先第一眼要注意，是**FactoryBean**接口而不是**BeanFactory**接口。一般情况下，Spring通过反射机制利用bean的class属性指定实现类来实例化bean ，实例化bean过程比较复杂。FactoryBean接口就是为了简化此过程，把bean的实例化定制逻辑下发给使用者。

在该接口中还定义了以下3个方法。

T getObject()：返回由FactoryBean创建的bean实例，如果isSingleton()返回true，则该实例会放到Spring容器中单实例缓存池中。

boolean isSingleton()：返回由FactoryBean创建的bean实例的作用域是singleton还是prototype。

Class<T> getObjectType()：返回FactoryBean创建的bean类型。

**注意一点：通过Spring容器的getBean()方法返回的不是FactoryBean本身，而是FactoryBean#getObject()方法所返回的对象，相当于FactoryBean#getObject()代理了getBean()方法。如果希望获取CarFactoryBean的实例，则需要在使用getBean(beanName) 方法时在beanName前显示的加上 "&" 前缀。**

一个例子：

实体类：

```java
public class Fruit {

	private String name;
	private String color;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getColor() {
		return color;
	}

	public void setColor(String color) {
		this.color = color;
	}

	@Override
	public String toString() {
		return "Fruit{" +
				"name='" + name + '\'' +
				", color='" + color + '\'' +
				'}';
	}
}
```

自定义FactoryBean:

```java
@Component
public class FruitFactoryBean implements FactoryBean<Fruit> {

	@Override
	public Fruit getObject() throws Exception {
		Fruit fruit = new Fruit();
		fruit.setColor("red");
		fruit.setName("apple");
		return fruit;
	}

	@Override
	public Class<?> getObjectType() {
		return Fruit.class;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}
}
```

测试类：

```java
@SpringBootTest(classes = Application.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class FruitFactoryBeanTest {

	@Autowired
	private FruitFactoryBean fruitFactoryBean;

	@Autowired
	private ApplicationContext applicationContext;

	@Test
	public void getObject() throws Exception {
		//直接通过#getObject获取实例
		Fruit apple = fruitFactoryBean.getObject();
		System.out.println(apple.toString());
		//通过Spring上下文获取实例
		Fruit fruit = (Fruit) applicationContext.getBean("fruitFactoryBean");
		System.out.println(fruit);
		//获取FruitFactoryBean自身的实例
		FruitFactoryBean bean = (FruitFactoryBean) applicationContext.getBean("&fruitFactoryBean");
		System.out.println(bean);
	}

}
```

结果：

控制台输出：

```
Fruit{name='apple',color='red'}
Fruit{name='apple',color='red'}
org.throwable.factorybean.FruitFactoryBean@5aa360ea
```

结果和预期一样，通过ApplicationContext#getBean(beanName)获取到的实际上是FactoryBean#getObject的实例，ApplicationContext#getBean(“&” + beanName)获取到的才是FruitFactoryBean本身的实例。

### 7、ApplicationListener接口

**ApplicationListener**是一个接口，里面只有一个onApplicationEvent(E event)方法，这个泛型E必须是**ApplicationEvent**的子类，而ApplicationEvent是Spring定义的事件，继承于EventObject，构造要求必须传入一个Object类型的source，这个source可以作为一个存储对象。将会在ApplicationListener的onApplicationEvent里面得到回调。**如果在上下文中部署一个实现了ApplicationListener接口的bean，那么每当在一个ApplicationEvent发布到 ApplicationContext时，这个bean得到通知。其实这就是标准的Oberver设计模式。**另外，ApplicationEvent的发布由ApplicationContext通过#publishEvent方法完成。其实这个实现从原理和代码上看都有点像Guava的eventbus。

贴一个例子：

EmailEvent:

```java
public class EmailEvent extends ApplicationEvent {

	private String author;
	private String content;
	private String date;

	public EmailEvent(Object source, String author, String content, String date) {
		super(source);
		this.author = author;
		this.content = content;
		this.date = date;
	}

	public String getAuthor() {
		return author;
	}

	public void setAuthor(String author) {
		this.author = author;
	}

	public String getContent() {
		return content;
	}

	public void setContent(String content) {
		this.content = content;
	}

	public String getDate() {
		return date;
	}

	public void setDate(String date) {
		this.date = date;
	}
}
```

EmailApplicationListener：

```java
@Component
public class EmailApplicationListener implements ApplicationListener<EmailEvent> {

	@Override
	public void onApplicationEvent(EmailEvent event) {
		System.out.println("EmailApplicationListener callback!!");
		System.out.println("EmailEvent --> source: " + event.getSource());
		System.out.println("EmailEvent --> author: " + event.getAuthor());
		System.out.println("EmailEvent --> content: " + event.getContent());
		System.out.println("EmailEvent --> date: " + event.getDate());
	}
}
```

测试类：

```java
@SpringBootTest(classes = Application.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class EmailApplicationListenerTest {

	@Autowired
	private ApplicationContext applicationContext;

	@Test
	public void onApplicationEvent() throws Exception {
		applicationContext.publishEvent(new EmailEvent("this is source",
				"throwable","here is emailEvent","2017-5-16"));
	}

}
```

控制台输出：

```
EmailApplicationListener callback!!
EmailEvent --> source: this is source
EmailEvent --> author: throwable
EmailEvent --> content: here is emailEvent
EmailEvent --> date: 2017-5-16
```
### 8、SmartLifecycle接口

内容转自[SmartLifecycle](http://blog.csdn.net/catoop/article/details/71274561)

在使用Spring开发时，我们都知道，所有bean都交给Spring容器来统一管理，其中包括没一个bean的加载和初始化。 
有时候我们需要在Spring加载和初始化所有bean后，接着执行一些任务或者启动需要的异步服务，这样我们可以使用 SmartLifecycle 来做到。

SmartLifecycle 是一个接口。当Spring容器加载所有bean并完成初始化之后，会接着回调实现该接口的类中对应的方法（start()方法）。 
如下是一个实例： 
我在对应的方法上注上了明确的说明，注意看一下。

```java
package com.shanhy.sboot;

import org.springframework.context.SmartLifecycle;
import org.springframework.stereotype.Component;

/**
 * SmartLifecycle测试
 *
 * @author 单红宇(CSDN CATOOP)
 * @create 2017年5月6日
 */
@Component
public class TestSmartLifecycle implements SmartLifecycle {

    private boolean isRunning = false;

    /**
     * 1. 我们主要在该方法中启动任务或者其他异步服务，比如开启MQ接收消息<br/>
     * 2. 当上下文被刷新（所有对象已被实例化和初始化之后）时，将调用该方法，默认生命周期处理器将检查每个SmartLifecycle对象的isAutoStartup()方法返回的布尔值。
     * 如果为“true”，则该方法会被调用，而不是等待显式调用自己的start()方法。
     */
    @Override
    public void start() {
        System.out.println("start");

        // 执行完其他业务后，可以修改 isRunning = true
        isRunning = true;
    }

    /**
     * 如果工程中有多个实现接口SmartLifecycle的类，则这些类的start的执行顺序按getPhase方法返回值从小到大执行。<br/>
     * 例如：1比2先执行，-1比0先执行。 stop方法的执行顺序则相反，getPhase返回值较大类的stop方法先被调用，小的后被调用。
     */
    @Override
    public int getPhase() {
        // 默认为0
        return 0;
    }

    /**
     * 根据该方法的返回值决定是否执行start方法。<br/> 
     * 返回true时start方法会被自动执行，返回false则不会。
     */
    @Override
    public boolean isAutoStartup() {
        // 默认为false
        return true;
    }

    /**
     * 1. 只有该方法返回false时，start方法才会被执行。<br/>
     * 2. 只有该方法返回true时，stop(Runnable callback)或stop()方法才会被执行。
     */
    @Override
    public boolean isRunning() {
        // 默认返回false
        return isRunning;
    }

    /**
     * SmartLifecycle子类的才有的方法，当isRunning方法返回true时，该方法才会被调用。
     */
    @Override
    public void stop(Runnable callback) {
        System.out.println("stop(Runnable)");

        // 如果你让isRunning返回true，需要执行stop这个方法，那么就不要忘记调用callback.run()。
        // 否则在你程序退出时，Spring的DefaultLifecycleProcessor会认为你这个TestSmartLifecycle没有stop完成，程序会一直卡着结束不了，等待一定时间（默认超时时间30秒）后才会自动结束。
        // PS：如果你想修改这个默认超时时间，可以按下面思路做，当然下面代码是springmvc配置文件形式的参考，在SpringBoot中自然不是配置xml来完成，这里只是提供一种思路。
        // <bean id="lifecycleProcessor" class="org.springframework.context.support.DefaultLifecycleProcessor">
        //      <!-- timeout value in milliseconds -->
        //      <property name="timeoutPerShutdownPhase" value="10000"/>
        // </bean>
        callback.run();

        isRunning = false;
    }

    /**
     * 接口Lifecycle的子类的方法，只有非SmartLifecycle的子类才会执行该方法。<br/>
     * 1. 该方法只对直接实现接口Lifecycle的类才起作用，对实现SmartLifecycle接口的类无效。<br/>
     * 2. 方法stop()和方法stop(Runnable callback)的区别只在于，后者是SmartLifecycle子类的专属。
     */
    @Override
    public void stop() {
        System.out.println("stop");

        isRunning = false;
    }

}
```

### 9、SmartInitializingSingleton接口

Spring4.1后添加的接口，作用是:当所有的singleton的bean都初始化完了之后才会回调这个接口。

接口的描述见源码:

```java
public interface SmartInitializingSingleton {

    /**
     * Invoked right at the end of the singleton pre-instantiation phase,
     * with a guarantee that all regular singleton beans have been created
     * already. {@link ListableBeanFactory#getBeansOfType} calls within
     * this method won't trigger accidental side effects during bootstrap.
     * <p><b>NOTE:</b> This callback won't be triggered for singleton beans
     * lazily initialized on demand after {@link BeanFactory} bootstrap,
     * and not for any other bean scope either. Carefully use it for beans
     * with the intended bootstrap semantics only.
     */
    void afterSingletonsInstantiated();

}
```

Spring组件中很多类都实现了这个接口,例如:RabbitListenerAnnotationBeanPostProcessor。

## Springboot扩展接口

### 1、ApplicationRunner接口和CommandLineRunner接口

在日常开发的时候，我们可能遇到这种情景：Spring容器启动完成的时候，我们需要程序做某些操作。这两个接口就是为此而生。CommandLineRunner和ApplicationRunner接口的**执行时机为容器启动完成的时候（由SpringApplication的#run方法跟踪进去，发现在调用#refreshContext方法调用之后调用#afterRefresh，#afterRefresh方法就是遍历并且执行所有的ApplicationRunner和CommandLineRunner接口#run方法）。**这两个接口中有一个run方法，我们只需要实现这个方法即可。这两个接口的不同之处在于：ApplicationRunner中run方法的参数为ApplicationArguments，而CommandLineRunner接口中run方法的参数为String数组。Springboot允许多个CommandLineRunner和ApplicationRunner存在并且由**@Order**注解决定执行顺序，**@Order的value值越小，执行的优先度越高。(使用@Order注解生效的条件是该类必须为Spring的构件，例如使用@Component作用在类上)。**

例子：

```java
@Component  
public class TestImplApplicationRunner implements ApplicationRunner {  

    @Override  
    public void run(ApplicationArguments args) throws Exception {  
        System.out.println(args);  
        System.out.println("这个是测试ApplicationRunner接口");  
    }  
}  
```

```java
@Component  
public class TestImplCommandLineRunner implements CommandLineRunner {  

    @Override  
    public void run(String... args) throws Exception {  
        System.out.println(args);  
        System.out.println("<<<<<<<<<<<<这个是测试CommandLineRunn接口>>>>>>>>>>>>>>");  
    }  
}  
```

