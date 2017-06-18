[TOC]

# SpringCloud与Docker微服务架构实战读书笔记

## 第一章要点

### 微服务架构的优点

* 易于开发和维护
* 单个微服务启动快
* 技术栈不受限
* 按需伸缩

### 微服务架构面临的挑战

* 运维要求较高
* 分布式固有的复杂性(如分布式事务等等...)
* 接口调整成本高(改动一个接口，所有调用这个接口的应用都要做调整)
* 重复劳动(多个微服务调用相同的功能模块，但是这个模块没有达到分解为一个微服务的程度，那么这些服务里面都会重复开发这部分功能代码)

### 微服务设计原则

* 单一职责原则
* 服务自治原则
* 轻量级通信机制(常用协议如REST、AMQP、STOMP、MQTT等)
* 微服务粒度(个人认为微服务的粒度是最难的一个原则，需要结合经验和项目的实际情况分析)



## 第二章要点

第二章主要是SpringCloud的简介。



## 第三章要点

### 使用SpringCloud实战微服务

* ### 工具以及版本

Springboot: 1.5.2.RELEASE

SpringCloud: Dalston.SR1

* ### 服务提供者与服务消费者

|  名词   |           定义           |
| :---: | :--------------------: |
| 服务提供者 | 服务的被调用方(即为其他服务提供服务的服务) |
| 服务消费者 |   服务的调用方(即依赖其他服务的服务)   |

* ### 本章例子:电影售票系统

![services-ref](..\SpringCloud与Docker微服务架构实战读书笔记\services-ref.png)



* ### 编写服务提供者microservice-simple-provider-user

**项目主要使用到JPA、H2数据库**

主pom.xml:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.throwable</groupId>
    <artifactId>springcloud-seed</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>microservice-simple-provider-user</module>
    </modules>
    <name>springcloud-seed</name>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.2.RELEASE</version>
    </parent>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Dalston.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>

    </dependencies>

    <build>
        <finalName>springcloud-seed</finalName>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>

            </plugin>
        </plugins>
    </build>
</project>

```

项目pom.xml:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <parent>
        <artifactId>springcloud-seed</artifactId>
        <groupId>org.throwable</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>microservice-simple-provider-user</artifactId>
    <packaging>jar</packaging>
    <name>microservice-simple-provider-user</name>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
        </dependency>

    </dependencies>
    <build>
        <finalName>microservice-simple-provider-user</finalName>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```

创建User实体:

```java
@Entity
public class User {

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	private Long id;

	@Column
	private String username;

	@Column
	private String name;

	@Column
	private Integer age;

	@Column
	private BigDecimal balance;

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Integer getAge() {
		return age;
	}

	public void setAge(Integer age) {
		this.age = age;
	}

	public BigDecimal getBalance() {
		return balance;
	}

	public void setBalance(BigDecimal balance) {
		this.balance = balance;
	}
}
```

创建DAO:

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}

```

创建controller:

```java
@RestController
public class UserController {

	@Autowired
	private UserRepository userRepository;

	@GetMapping("/{id}")
	public User findById(@PathVariable("id") Long id) {
		return userRepository.findOne(id);
	}
}
```

Springboot主函数:

```java
@SpringBootApplication
public class ProviderUserApplication {

	public static void main(String[] args) {
		SpringApplication.run(ProviderUserApplication.class, args);
	}
}
```

* ### 编写服务消费者microservice-simple-consumer-movie

**项目主要使用到RestTemplate**

项目的pom.xml:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <parent>
        <artifactId>springcloud-seed</artifactId>
        <groupId>org.throwable</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>microservice-simple-consumer-movie</artifactId>
    <packaging>jar</packaging>
    <name>microservice-simple-consumer-movie</name>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <build>
        <finalName>microservice-simple-consumer-movie</finalName>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```

pojo:

```java
public class User {

	private Long id;
	private String username;
	private String name;
	private Integer age;
	private BigDecimal balance;

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Integer getAge() {
		return age;
	}

	public void setAge(Integer age) {
		this.age = age;
	}

	public BigDecimal getBalance() {
		return balance;
	}

	public void setBalance(BigDecimal balance) {
		this.balance = balance;
	}
}
```

controller:

```java
@RestController
public class MovieController {

	@Autowired
	private RestTemplate restTemplate;

	@GetMapping("/user/{id}")
	public User findById(@PathVariable("id") Long id) {
		return restTemplate.getForObject("http://localhost:8081/" + id, User.class);
	}
}
```

Springboot主函数:

```java
@SpringBootApplication
public class ConsumerMovieApplication {

	@Bean
	public RestTemplate restTemplate() {
		return new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(ConsumerMovieApplication.class, args);
	}
}
```

用IDEA的restclient工具调用:

![get-user](..\SpringCloud与Docker微服务架构实战读书笔记\get-user.png)

说明电影微服务可以通过RestTemplate正常调用用户微服务的接口。

### 为项目整合Springboot Actuator

Springboot Actuator提供很多监控端点，调用的方式是:http://{ip}:{port}/{endpoint}，可以从返回值了解当前应用程序的运行情况。

Actuator提供的端点如下:

|     端点      |                    描述                    | HTTP方法 |
| :---------: | :--------------------------------------: | :----: |
| autoconfig  |                显示自动配置的信息                 |  GET   |
|    beans    |         显示应用程序上下文所有的Spring bean          |  GET   |
| configprops |   显示所有的@ConfigurationProperties的配置属性列表   |  GET   |
|    dump     |                显示线程活动的快照                 |  GET   |
|     env     |                显示应用的环境变量                 |  GET   |
|   health    |  显示应用程序的健康指标，这些值由HealthIndicator的实现类提供   |  GET   |
|    info     |    显示应用的信息，可以使用info.*属性自定义info端点公开的数据    |  GET   |
|  mappings   |                显示所有的URL路径                |  GET   |
|   metrics   |               显示应用的度量标准信息                |  GET   |
|  shutdown   | 关闭应用(默认情况下不启用，如果需要启动，需要设置endpoints.shutdown.enabled=true) |  POST  |
|    trace    |                  显示跟踪信息                  |  GET   |

**测试例子**

项目microservice-simple-provider-user添加maven依赖:

```xml
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

在Springboot 1.5.2.RELEASE版本中，部分端点设置了访问权限，测试时候通过配置下面属性关闭权限认证:

```yaml
management:
  security:
    enabled: false
```

![acturator-health-test](..\SpringCloud与Docker微服务架构实战读书笔记\acturator-health-test.png)

配置自定义的info属性:

```yaml
info:
  app:
    name: @artifactId@
    encoding: @project.build.sourceEncoding@
    java:
      source: ${java.version}
      target: ${java.version}
```

![acturator-health-test](..\SpringCloud与Docker微服务架构实战读书笔记\acturator-info-test.png)

### 项目遗留问题

```java
    @GetMapping("/user/{id}")
	public User findById(@PathVariable("id") Long id) {
		return restTemplate.getForObject("http://localhost:8081/" + id, User.class);
	}
```

在MovieController中存在硬编码问题，一个比较简单的做法是把服务提供者的接口写成配置放进去yaml配置文件里面再自动注入获取，但是这样做又会出现下面问题:

* 适用场景有局限性：如果服务提供者的网络地址(host、port)发生变化，将会影响到服务消费者。
* 无法动态伸缩：生产环境中，每个微服务一般会部署多个实例，实现容灾和负载均衡，硬编码即使做成配置也是无法适应这种需求。



## 第四章要点



