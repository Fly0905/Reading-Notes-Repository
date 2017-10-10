[TOC]

# Hystrix使用笔记
# 一、简介

在大中型分布式系统中，通常系统很多依赖。在高并发访问下,这些依赖的稳定性与否对系统的影响非常大,但是依赖有很多不可控问题:如网络连接缓慢，资源繁忙，暂时不可用，服务脱机等。在复杂的分布式架构的应用程序有很多的依赖，都会不可避免地在某些时候失败。高并发的依赖失败时如果没有隔离措施，当前应用服务就有被拖垮的风险。一般来说，随着服务依赖数量的变多，服务不稳定的概率会成指数性提高。例如：

一个依赖30个SOA服务的系统,每个服务99.99%可用。  
99.99%的30次方 ≈ 99.7%，  
0.3% 意味着一亿次请求 会有 3,000,00次失败， 
换算成时间大约每月有2个小时服务不稳定。
解决这个问题的方案是对依赖进行隔离。Hystrix就是处理依赖隔离的框架,同时也是可以帮我们做依赖服务的治理和监控。Hystrix英文翻译就是豪猪，豪猪科动物以棘刺闻名，棘刺有保护御敌作用。Netflix自称Hystrix在其内部的使用规模如下：

The Netflix API processes 10+ billion HystrixCommand executions per day using thread isolation.   
Each API instance has 40+ thread-pools with 5-20 threads in each (most are set to 10).
[Netflix API每天使用线程隔离处理100+亿次的HystrixCommand。  
每个API实例都有40多个线程池，每个线程池都有5-20个线程(大多数都设置为10个线程)。]
PS：本文使用VScode编写，如果需要使用[TOC]目录导航，可以使用Typora打开。

# 二、Hystrix依赖隔离的原理

Hystrix使用命令模式(Command)包装依赖调用逻辑，每个命令在单独线程中/信号授权下执行。
可配置依赖调用超时时间,超时时间一般设为比99.5%平均时间略高即可。当调用超时时，直接返回或执行fallback（降级）逻辑。
为每个依赖提供一个小的线程池（或信号），如果线程池已满调用将被立即拒绝，默认不采用排队（使用SynchronousQueue和拒绝策略），加速失败判定时间(快速失败)。
依赖调用结果分：成功，失败（抛出异常），超时，线程拒绝，短路。请求失败(异常，拒绝，超时，短路)时执行fallback(降级)逻辑。
提供熔断器组件（下一个小节详细说明）。
提供近实时依赖的统计和监控（详细的metrics（度量）信息）。

# 三、Hystrix熔断机制

下面简单说明一下Hystrix的熔断机制。是否开启熔断器主要由依赖调用的错误比率决定的，依赖调用的错误比率=请求失败数/请求总数。Hystrix中断路器打开的默认请求错误比率为50%（这里暂时称为请求错误率），还有一个参数，用于设置在一个滚动窗口中，打开断路器的最少请求数（这里暂时称为滚动窗口最小请求数），这里举个具体的例子：如果滚动窗口最小请求数为20，在一个窗口内（比如10秒，统计滚动窗口的时间可以设置，见下面的参数详解），收到19个请求，即使这19个请求都失败了，此时请求错误率高达95%，但是断路器也不会打开。对于被熔断的请求，并不是永久被切断，而是被暂停一段时间（默认是5000ms）之后，允许部分请求通过，若请求都是健康的（ResponseTime<250ms）则对请求健康恢复（取消熔断），如果不是健康的，则继续熔断。（这里很容易出现一种错觉：多个请求失败但是没有触发熔断。这是因为在一个滚动窗口内的失败请求数没有达到打开断路器的最少请求数）

# 四、Hystrix配置参数详细说明

内置全局默认值（Global default from code）
如果下面3种都没有设置，默认是使用此种，后面用"默认值"代指这种。

动态全局默认属性（Dynamic global default property）
可以通过属性配置来更改全局默认值，后面用"默认属性"代指这种。

内置实例默认值（Instance default from code）
在代码中，设置的属性值，后面用"实例默认"来代指这种。

动态配置实例属性（Dynamic instance property）
可以针对特定的实例，动态配置属性值，来代替前面三种，后面用"实例属性"来代指这种。

优先级：1 < 2 < 3 < 4。 这些配置基本上可以从com.netflix.hystrix.HystrixCommandProperties和com.netflix.hystrix.HystrixThreadPoolProperties查看。

## 基础属性配置

#### CommandGroup

CommandGroup是每个命令最少配置的必选参数，在不指定ThreadPoolKey的情况下，字面值用于对不同依赖的线程池/信号区分，也就是在不指定ThreadPoolKey的情况下,CommandGroup用于指定线程池的隔离。命令分组用于对依赖操作分组，便于统计、汇总等。

* 实例属性：com.netflix.hystrix.HystrixCommandGroupKey
* 实例配置：HystrixCommand.Setter().withGroupKey (HystrixCommandGroupKey.Factory.asKey("Group"));
* 注解使用：@HystrixCommand(groupKey = "Group")

#### CommandKey

CommandKey是作为依赖命名，一般来说每个CommandKey代表一个依赖抽象，相同的依赖要使用相同的CommandKey名称。依赖隔离的根本就是对相同CommandKey的依赖做隔离。不同的依赖隔离最好使用不同的线程池（定义不同的ThreadPoolKey）。从HystrixCommand源码的注释也可以看到CommandKey也用于对依赖操作统计、汇总等。

* 实例属性：com.netflix.hystrix.HystrixCommandKey
* 实例配置：HystrixCommand.Setter().andCommandKey(HystrixCommandKey.Factory.asKey("Key"))
* 注解使用：@HystrixCommand(commandKey = "Key")

#### ThreadPoolKey

ThreadPoolKey简单来说就是依赖隔离使用的线程池的键值。当对同一业务依赖做隔离时使用CommandGroup做区分，但是对同一依赖的不同远程调用如(一个是redis 一个是http)，可以使用HystrixThreadPoolKey做隔离区分。 虽然在业务上都是相同的组，但是需要在资源上做隔离时，可以使用HystrixThreadPoolKey区分。（对于每个不同的HystrixThreadPoolKey建议使用不同的CommandKey）

* 实例属性：com.netflix.hystrix.HystrixThreadPoolKey
* 实例配置：HystrixCommand.Setter().andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("Thread"))
* 注解使用：@HystrixCommand(threadPoolKey = "Thread")

## 命令属性配置

## (1)执行属性

#### execution.isolation.strategy

用于设置HystrixCommand执行的隔离策略，有两种选项：

THREAD —— 在固定大小线程池中，以单独线程执行，并发请求数受限于线程池大小。

SEMAPHORE —— 在调用线程中执行，通过信号量来限制并发量。

* 默认值：THREAD（ExecutionIsolationStrategy.THREAD）
* 可选值：THREAD，SEMAPHORE
* 默认属性：hystrix.command.default.execution.isolation.strategy
* 实例属性：hystrix.command.HystrixCommandKey.execution.isolation.strategy
* 实例配置： HystrixCommandProperties.Setter().withExecutionIsolationStrategy(ExecutionIsolationStrategy.THREAD)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "execution.isolation.strategy",value = "THREAD")})

#### execution.isolation.thread.timeoutInMilliseconds

设置调用者等待命令执行的超时限制，超过此时间，HystrixCommand被标记为TIMEOUT，并执行回退逻辑。

注意：超时会作用在HystrixCommand.queue()，即使调用者没有调用get()去获得Future对象。

* 默认值：1000（毫秒）
* 默认属性：hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds
* 实例属性：hystrix.command.HystrixCommandKey.execution.isolation.thread.timeoutInMilliseconds
* 实例配置：HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(int Value);
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")})

#### execution.timeout.enabled

设置HystrixCommand的执行是否有超时限制。

* 默认值：true
* 默认属性：hystrix.command.default.execution.timeout.enabled
* 实例属性：hystrix.command.HystrixCommandKey.execution.timeout.enabled
* 实例配置：HystrixCommandProperties.Setter().withExecutionTimeoutEnabled(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "execution.timeout.enabled",value = "true")})

#### execution.isolation.thread.interruptOnTimeout

设置HystrixCommand的执行是否在超时发生时被中断。使用线程隔离时，是否对命令执行超时的线程调用中断（Thread.interrupt()）操作。

* 默认值：true
* 默认属性：hystrix.command.default.execution.isolation.thread.interruptOnTimeout
* 实例属性：hystrix.command.HystrixCommandKey.execution.isolation.thread.interruptOnTimeout
* 实例配置：HystrixCommandProperties.Setter().withExecutionIsolationThreadInterruptOnTimeout(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "execution.isolation.thread.interruptOnTimeout",value = "true")})

#### execution.isolation.thread.interruptOnCancel

当HystrixCommand命令执行发生cancel事件后是否应该响应中断。

* 默认值：false
* 默认属性：hystrix.command.default.execution.isolation.thread.interruptOnCancel
* 实例属性：hystrix.command.HystrixCommandKey.execution.isolation.thread.interruptOnCancel
* 实例配置：HystrixCommandProperties.Setter().withExecutionIsolationThreadInterruptOnCancel(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "execution.isolation.thread.interruptOnCancel",value = "false")})

#### execution.isolation.semaphore.maxConcurrentRequests

设置当使用ExecutionIsolationStrategy.SEMAPHORE时，HystrixCommand执行方法允许的最大请求数。如果达到最大并发数时，后续请求会被拒绝。信号量应该是容器（比如Tomcat）线程池一小部分，不能等于或者略小于容器线程池大小，否则起不到保护作用。

* 默认值：10
* 默认属性：hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests
* 实例属性：hystrix.command.HystrixCommandKey.execution.isolation.semaphore.maxConcurrentRequests
* 实例配置：HystrixCommandProperties.Setter().withExecutionIsolationSemaphoreMaxConcurrentRequests(int Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "execution.isolation.semaphore.maxConcurrentRequests",value = "10")})

## (2)回退属性

下面的属性控制HystrixCommand.getFallback()执行。这些属性对ExecutionIsolationStrategy.THREAD和ExecutionIsolationStrategy.SEMAPHORE都有效。

#### fallback.isolation.semaphore.maxConcurrentRequests

设置调用线程产生的HystrixCommand.getFallback()方法的允许最大请求数目。如果达到最大并发数目，后续请求将会被拒绝，如果没有实现回退，则抛出异常。(这里需要注意一点，这个变量的命名不是很规范，它实际上对THREAD和SEMAPHORE两种隔离策略都生效)

* 默认值：10
* 默认属性：hystrix.command.default.fallback.isolation.semaphore.maxConcurrentRequests
* 实例属性：hystrix.command.HystrixCommandKey.fallback.isolation.semaphore.maxConcurrentRequests
* 实例配置：HystrixCommandProperties.Setter().withFallbackIsolationSemaphoreMaxConcurrentRequests(int Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests",value = "10")})

#### fallback.enabled

该属性决定当前的调用故障或者拒绝发生时，是否调用HystrixCommand.getFallback()。

* 默认值：true
* 默认属性：hystrix.command.default.fallback.enabled
* 实例属性：hystrix.command.HystrixCommandKey.fallback.enabled
* 实例配置：HystrixCommandProperties.Setter().withFallbackEnabled(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "fallback.enabled",value = "true")})

## (3)断路器（Circuit Breaker）属性配置

#### circuitBreaker.enabled

设置断路器是否生效。

* 默认值：true
* 默认属性：hystrix.command.default.circuitBreaker.enabled
* 实例属性：hystrix.command.HystrixCommandKey.circuitBreaker.enabled
* 实例配置：HystrixCommandProperties.Setter().withCircuitBreakerEnabled(boolean value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "circuitBreaker.enabled",value = "true")})

#### circuitBreaker.requestVolumeThreshold

设置在一个滚动窗口中，打开断路器的最少请求数。比如：如果值是20，在一个窗口内（比如10秒），收到19个请求，即使这19个请求都失败了，断路器也不会打开。(滚动窗口时间段的长度设置见下面的metrics.rollingStats.timeInMilliseconds)

* 默认值：20
* 默认属性：hystrix.command.default.circuitBreaker.requestVolumeThreshold
* 实例属性：hystrix.command.HystrixCommandKey.circuitBreaker.requestVolumeThreshold
* 实例配置：HystrixCommandProperties.Setter().withCircuitBreakerRequestVolumeThreshold(int Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "20")})

#### circuitBreaker.sleepWindowInMilliseconds

设置在断路器被打开，拒绝请求到再次尝试请求的时间间隔。

* 默认值：5000（毫秒）
* 默认属性：hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds
* 实例属性：hystrix.command.HystrixCommandKey.circuitBreaker.sleepWindowInMilliseconds
* 实例配置：HystrixCommandProperties.Setter().withCircuitBreakerSleepWindowInMilliseconds(int Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "5000")})

#### circuitBreaker.errorThresholdPercentage

设置打开断路器并启动回退逻辑的错误比率。（这个参数的效果受到circuitBreaker.requestVolumeThreshold和滚动时间窗口的时间长度影响）

* 默认值：50(%)
* 默认属性：hystrix.command.default.circuitBreaker.errorThresholdPercentage
* 实例属性：hystrix.command.HystrixCommandKey.circuitBreaker.errorThresholdPercentage
* 实例配置：HystrixCommandProperties.Setter().withCircuitBreakerErrorThresholdPercentage(int Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "50")})

#### circuitBreaker.forceOpen

如果该属性设置为true，强制断路器进入打开状态，将会拒绝所有的请求。该属性优先级比circuitBreaker.forceClosed高。

* 默认值：false
* 默认属性：hystrix.command.default.circuitBreaker.forceOpen
* 实例属性：hystrix.command.HystrixCommandKey.circuitBreaker.forceOpen
* 实例配置：HystrixCommandProperties.Setter().withCircuitBreakerForceOpen(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "circuitBreaker.forceOpen",value = "false")})

#### circuitBreaker.forceClosed

如果该属性设置为true，强制断路器进入关闭状态，将会允许所有的请求，无视错误率。

* 默认值：false
* 默认属性：hystrix.command.default.circuitBreaker.forceClosed
* 实例属性：hystrix.command.HystrixCommandKey.circuitBreaker.forceClosed
* 实例配置：HystrixCommandProperties.Setter().withCircuitBreakerForceClosed(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "circuitBreaker.forceClosed",value = "false")})

## (4)请求上下文属性配置

#### requestCache.enabled

设置HystrixCommand.getCacheKey()是否启用，由HystrixRequestCache通过请求缓存提供去重复数据功能。（请求结果缓存需要配合HystrixRequestContext使用，具体应用可以自行查阅）

* 默认值：true
* 默认属性：hystrix.command.default.requestCache.enabled
* 实例属性：hystrix.command.HystrixCommandKey.requestCache.enabled
* 实例配置：HystrixCommandProperties.Setter().withRequestCacheEnabled(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "requestCache.enabled",value = "true")})

#### requestLog.enabled

设置HystrixCommand执行和事件是否要记录日志到HystrixRequestLog。

* 默认值：true
* 默认属性：hystrix.command.default.requestLog.enabled
* 实例属性：hystrix.command.HystrixCommandKey.requestLog.enabled
* 实例配置：HystrixCommandProperties.Setter().withRequestLogEnabled(boolean Value)
* 注解使用：@HystrixCommand(commandProperties = { @HystrixProperty(name = "requestLog.enabled",value = "true")})

## (5)压缩器属性配置

HystrixCollapser主要用于请求的合并，在Hystrix注解体系中它有一个独立的注解@HystrixCollapser。

#### maxRequestsInBatch

设置触发批处理执行之前，在批处理中允许的最大请求数。

* 默认值：Integer.MAX_VALUE
* 默认属性：hystrix.collapser.default.maxRequestsInBatch
* 实例属性：hystrix.collapser.HystrixCollapserKey.maxRequestsInBatch
* 实例配置：HystrixCollapserProperties.Setter().withMaxRequestsInBatch(int Value)
* 注解使用：@HystrixCollapser(collapserProperties = { @HystrixProperty(name = "maxRequestsInBatch",value = "100")})

#### timerDelayInMilliseconds

设置批处理创建到执行之间的毫秒数，实际上就是这个时间间隔内发生的所有请求都会进行合并（为一个请求）。

* 默认值：10(毫秒)
* 默认属性：hystrix.collapser.default.timerDelayInMilliseconds
* 实例属性：hystrix.collapser.HystrixCollapserKey.timerDelayInMilliseconds
* 实例配置：HystrixCollapserProperties.Setter().withTimerDelayInMilliseconds(int Value)
* 注解使用：@HystrixCollapser(collapserProperties = { @HystrixProperty(name = "timerDelayInMilliseconds",value = "100")})

#### requestCache.enabled

设置请求缓存是否对HystrixCollapser.execute()和HystrixCollapser.queue()的调用起作用。（请求结果缓存需要配合HystrixRequestContext使用，具体应用可以自行查阅）

* 默认值：true
* 默认属性：hystrix.collapser.default.requestCache.enabled
* 实例属性：hystrix.collapser.HystrixCollapserKey.requestCache.enabled
* 实例配置：HystrixCollapserProperties.Setter().withRequestCacheEnabled(boolean Value)
* 注解使用：@HystrixCollapser(collapserProperties = { @HystrixProperty(name = "requestCache.enabled",value = "true")})

## (6)线程池属性配置

#### coreSize

设置核心线程池的大小（这个值和ThreadPoolExecutor的coreSize的含义不一样）。

* 默认值：10
* 默认属性：hystrix.threadpool.default.coreSize
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.coreSize
* 实例配置：HystrixThreadPoolProperties.Setter().withCoreSize(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "coreSize",value = "10")})

#### maximumSize

1.5.9新增属性，设置线程池最大值。这个是在不开始拒绝HystrixCommand的情况下支持的最大并发数。这个属性起作用的前提是设置了allowMaximumSizeToDrivergeFromCoreSize。1.5.9之前，核心线程池大小和最大线程池大小总是相同的。

* 默认值：10
* 默认属性：hystrix.threadpool.default.maximumSize
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.maximumSize
* 实例配置：HystrixThreadPoolProperties.Setter().withMaximumSize(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "maximumSize",value = "10")})

#### maxQueueSize

设置BlockingQueue最大的队列值。如果设置为-1，那么使用SynchronousQueue，否则正数将会使用LinkedBlockingQueue。如果需要去除这些限制，允许队列动态变化，可以参考queueSizeRejectionThreshold属性。 修改SynchronousQueue和LinkedBlockingQueue需要重启。

* 默认值：-1
* 默认属性：hystrix.threadpool.default.maxQueueSize
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.maxQueueSize
* 实例配置：HystrixThreadPoolProperties.Setter().withMaxQueueSize(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "maxQueueSize",value = "10")})

#### queueSizeRejectionThreshold

设置队列拒绝的阈值----一个人为设置的拒绝访问的最大队列值，即使当前队列元素还没达到maxQueueSize。 当将一个线程放入队列等待执行时，HystrixCommand使用该属性。注意：如果maxQueueSize设置为-1，该属性不可用。

* 默认值：5
* 默认属性：hystrix.threadpool.default.queueSizeRejectionThreshold
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.queueSizeRejectionThreshold
* 实例默认的设置：HystrixThreadPoolProperties.Setter().withQueueSizeRejectionThreshold(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "queueSizeRejectionThreshold",value = "5")})

#### keepAliveTimeMinutes

设置存活时间，单位分钟。如果coreSize小于maximumSize，那么该属性控制一个线程从实用完成到被释放的时间。

* 默认值：1
* 默认属性：hystrix.threadpool.default.keepAliveTimeMinutes
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.keepAliveTimeMinutes
* 实例配置：HystrixThreadPoolProperties.Setter().withKeepAliveTimeMinutes(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "keepAliveTimeMinutes",value = "1")})

#### allowMaximumSizeToDivergeFromCoreSize

在1.5.9中新增的属性。该属性允许maximumSize起作用。属性值可以等于或者大于coreSize值，设置coreSize小于maximumSize的线程池能够支持maximumSize的并发数，但是会将不活跃的线程返回到系统中去。（详见KeepAliveTimeMinutes） 
* 默认值：false 
* 默认属性：hystrix.threadpool.default.allowMaximumSizeToDivergeFromCoreSize
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.allowMaximumSizeToDivergeFromCoreSize 
 * 实例配置：HystrixThreadPoolProperties.Setter().withAllowMaximumSizeToDivergeFromCoreSize(boolean Value)

## (7)度量属性配置
PS:不知道什么原因，计量属性的配置都是放在了线程池配置里面。可能是由于线程池隔离是计量属性隔离的基准。

#### metrics.rollingStats.timeInMilliseconds

设置统计的滚动窗口的时间段大小。该属性是线程池保持指标时间长短。
* 默认值：10000（毫秒）
* 默认属性：hystrix.threadpool.default.metrics.rollingStats.timeInMilliseconds
* 实例属性：hystrix.threadpool.HystrixThreadPoolKey.metrics.rollingStats.timeInMilliseconds
* 实例配置：HystrixThreadPoolProperties.Setter().withMetricsRollingStatisticalWindowInMilliseconds(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds",value = "10000")})

#### metrics.rollingStats.numBuckets

设置滚动的统计窗口被分成的桶（bucket）的数目。注意："metrics.rollingStats.timeInMilliseconds % metrics.rollingStats.numBuckets == 0"必须为true，否则会抛出异常。
* 默认值：10
* 可能的值：任何能被metrics.rollingStats.timeInMilliseconds整除的值。
* 默认属性：hystrix.threadpool.default.metrics.rollingStats.numBuckets
* 实例属性：hystrix.threadpool.HystrixThreadPoolProperties.metrics.rollingStats.numBuckets
* 实例配置：HystrixThreadPoolProperties.Setter().withMetricsRollingStatisticalWindowBuckets(int Value)
* 注解使用： @HystrixCommand(threadPoolProperties = {@HystrixProperty(name = "metrics.rollingStats.numBuckets",value = "10")})

# 五、Hystrix基于编程式和注解使用详解

## 编程式使用Hystrix

### （1）HystrixCommand vs HystrixObservableCommand
想要编程式使用Hystrix，只需要继承`HystrixCommand`或`HystrixObservableCommand`，这两者的主要区别是：
* `HystrixCommand`的命令逻辑写在run()；`HystrixObservableCommand`的命令逻辑写在construct()。
* `HystrixCommand`的run()是由新创建的线程执行；`HystrixObservableCommand`的construct()是由调用程序线程执行。
* `HystrixCommand`一个实例只能向调用程序发送（emit）单条数据，也就是run()只能返回一个结果；
`HystrixObservableCommand`一个实例可以顺序发送多条数据，顺序调用多个onNext()，便实现了向调用程序发送多条数据，甚至还能发送一个范围的数据集。

### （2）4个命令执行方法
execute()、queue()、observe()、toObservable()这4个方法用来触发执行run()/construct()，一个实例只能执行一次这4个方法，特别说明的是`HystrixObservableCommand`没有execute()和queue()，`HystrixCommand`对应run()，`HystrixObservableCommand`对应construct()。这4个方法的主要区别如下：
* execute()：以同步堵塞方式执行run()。`HystrixCommand`实例调用execute()后，hystrix先创建一个新线程运行run()，接着调用程序要在execute()调用处一直堵塞着，直到run()运行完成。
* queue()：以异步非堵塞方式执行run()。`HystrixCommand`实例调用queue()就直接返回一个Future对象，同时hystrix创建一个新线程运行run()，调用程序通过Future.get()拿到run()的返回结果，而Future.get()是堵塞执行的。
* observe()：**事件注册前**执行run()/construct()。第一步是事件注册前，先调用observe()自动触发执行run()/construct()（如果继承的是`HystrixCommand`，hystrix将创建新线程非堵塞执行run()；如果继承的是`HystrixObservableCommand`，将以调用程序线程堵塞执行construct()），第二步是从observe()返回后调用程序调用subscribe()完成事件注册，如果run()/construct()执行成功则触发onNext()和onCompleted()，如果执行异常则触发onError()。
* toObservable()：**事件注册后**执行run()/construct()。第一步是事件注册前，一调用toObservable()就直接返回一个Observable<T>对象，第二步调用subscribe()完成事件注册后自动触发执行run()/construct()（如果继承的是`HystrixCommand`，hystrix将创建新线程非堵塞执行run()，调用程序不必等待run()；如果继承的是`HystrixObservableCommand`，将以调用程序线程堵塞执行construct()，调用程序等待construct()执行完才能继续往下走），如果run()/construct()执行成功则触发onNext()和onCompleted()，如果执行异常则触发onError()。

### （3）fallback（降级）
使用fallback机制很简单，继承`HystrixCommand`只需重写`getFallback()`，继承`HystrixObservableCommand`只需重写`resumeWithFallback()`。fallback实际流程是当run()/construct()被触发执行时或执行中发生错误时，将转向执行getFallback()/resumeWithFallback()。调用程序可以通过isResponseFromFallback()查询结果是由run()/construct()还是getFallback()/resumeWithFallback()返回的。下面的情况会触发fallback：
* 非HystrixBadRequestException异常：当抛出HystrixBadRequestException时，调用程序可以捕获异常，此时不会触发fallback，而其他异常则会触发fallback，调用程序将获得fallback逻辑的返回结果。
* run()/construct()运行超时：执行命令的方法超时，将会触发fallback。
* 熔断器开启：当熔断器处于开启的状态，将会触发fallback。
* 线程池/信号量已满：当线程池/信号量已满的状态，将会触发fallback。

**这里需要注意：触发了降级逻辑不一定是熔断器开启，但是熔断器开启一定会执行降级逻辑**。

### 例子1（四种命令执行方法的结果获取）：
```
import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import rx.Observable;
import rx.Observer;

import java.util.concurrent.Future;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/9/28 15:32
 */
public class HelloWorldCommand extends HystrixCommand<String> {

    private final String name;

    public HelloWorldCommand(String name) {
        //最小配置,指定groupKey
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("helloWorldGroup"))
                //commonKey代表一个依赖抽象,相同的依赖要用相同的commonKey,依赖隔离的根本就是依据commonKey进行隔离
        .andCommandKey(HystrixCommandKey.Factory.asKey("helloWorld")));
        this.name = name;
    }

    @Override
    protected String run() throws Exception {
        return "Hello " + name + ",current thread:" + Thread.currentThread().getName();
    }

    public static void main(String[] args) throws Exception {
        HelloWorldCommand command = new HelloWorldCommand("doge");
        //1、同步调用
        String result = command.execute();
        System.out.println("Sync call result --> " + result);

        //2、异步调用
        command = new HelloWorldCommand("doge async");
        Future<String> future = command.queue();
        result = future.get();
        System.out.println("Async call result --> " + result);

        //3.1、注册观察者事件订阅 -- 事件注册前执行
        Observable<String> observable = new HelloWorldCommand("doge observable").observe();

        observable.subscribe(result1 -> System.out.println("Observable call result --> " + result1));

        //3.2、注册完整执行生命周期事件 -- 事件注册前执行
        observable.subscribe(new Observer<String>() {
            @Override
            public void onCompleted() {
                //onNext/onError完成之后最后回调
                System.out.println("Execute onCompleted");
            }

            @Override
            public void onError(Throwable throwable) {
                // 当产生异常时回调
                System.out.println("Execute error");
                throwable.printStackTrace();
            }

            @Override
            public void onNext(String s) {
                // 获取结果后回调
                System.out.println("Execute onNext --> " + s);
            }
        });

        //4、注册观察者事件订阅 -- 事件注册后执行
		command = new HelloWorldCommand("doge toObservable");
		Observable<String> toObservable = command.toObservable();
		toObservable.subscribe(new Observer<String>() {
			@Override
			public void onCompleted() {
				//onNext/onError完成之后最后回调
				System.out.println("Execute onCompleted");
			}

			@Override
			public void onError(Throwable throwable) {
				// 当产生异常时回调
				System.out.println("Execute error");
				throwable.printStackTrace();
			}

			@Override
			public void onNext(String s) {
				// 获取结果后回调
				System.out.println("Execute onNext --> " + s);
			}
		});

		//异步执行需要时间，先阻塞主线程
		Thread.sleep(5000);
	}
}
```
### 控制台输出：
```
Sync call result --> Hello doge,current thread:hystrix-helloWorldGroup-1
Async call result --> Hello doge async,current thread:hystrix-helloWorldGroup-2
Observable call result --> Hello doge observable,current thread:hystrix-helloWorldGroup-3
Execute onNext --> Hello doge observable,current thread:hystrix-helloWorldGroup-3
Execute onCompleted
Execute onNext --> Hello doge toObservable,current thread:hystrix-helloWorldGroup-4
Execute onCompleted
```

### 例子2（超时降级）：
```
import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixCommandProperties;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/9/28 15:32
 */
public class HelloWorldTimeoutCommand extends HystrixCommand<String> {

    private final String name;

    public HelloWorldTimeoutCommand(String name) {
        //最小配置,指定groupKey
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("helloWorldGroup"))
                //指定超时时间为500ms
        .andCommandPropertiesDefaults(HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(500))
                //commonKey
        .andCommandKey(HystrixCommandKey.Factory.asKey("helloWorldTimeout")));
        this.name = name;
    }

    @Override
    protected String run() throws Exception {
		System.out.println("HelloWorldTimeoutCommand --> "+ Thread.currentThread().getName());
		Thread.sleep(1000);
        return "Hello " + name + ",current thread:" + Thread.currentThread().getName();
    }

    @Override
    protected String getFallback() {
        return "fallback!";
    }

    public static void main(String[] args) throws Exception {
        HelloWorldTimeoutCommand command = new HelloWorldTimeoutCommand("doge");
        //超时执行getFallback
        System.out.println(command.execute());
    }
}
```
### 控制台输出：
```
HelloWorldTimeoutCommand --> hystrix-helloWorldGroup-1
fallback!
```

### 例子3（触发熔断器熔断）：
```
import com.netflix.hystrix.*;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/9/28 15:32
 */
public class HelloWorldBreakerCommand extends HystrixCommand<String> {

	private final String name;

	public HelloWorldBreakerCommand(String name) {
		//最小配置,指定groupKey
		super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("helloWorldGroup"))
				.andThreadPoolPropertiesDefaults(
						HystrixThreadPoolProperties.Setter()
								.withCoreSize(500))
				.andCommandPropertiesDefaults(
						HystrixCommandProperties.Setter()
								.withCircuitBreakerEnabled(true)
								.withCircuitBreakerErrorThresholdPercentage(50)
								.withCircuitBreakerRequestVolumeThreshold(3)
								.withExecutionTimeoutInMilliseconds(1000))
				//commonKey
				.andCommandKey(HystrixCommandKey.Factory.asKey("helloWorldBreaker")));
		this.name = name;
	}

	@Override
	protected String run() throws Exception {
		System.out.println("RUNNABLE --> " + name);
		Integer num = Integer.valueOf(name);
		if (num % 2 == 0 && num < 10) {
			return "Hello " + name + ",current thread:" + Thread.currentThread().getName();
		} else {
			Thread.sleep(1500);
			return name;
		}
	}

	@Override
	protected String getFallback() {
		return "FALLBACK --> !";
	}

	public static void main(String[] args) throws Exception {
		for (int i = 0; i < 50; i++) {
			try {
				System.out.println(new HelloWorldBreakerCommand(String.valueOf(i)).execute());
			} catch (Exception e) {
				e.printStackTrace();
			}
		}

		Thread.sleep(Integer.MAX_VALUE);
	}
}
```
### 控制台输出：
```
RUNNABLE --> 0
Hello 0,current thread:hystrix-helloWorldGroup-1
RUNNABLE --> 1
FALLBACK --> !
RUNNABLE --> 2
Hello 2,current thread:hystrix-helloWorldGroup-3
RUNNABLE --> 3
FALLBACK --> !
RUNNABLE --> 4
Hello 4,current thread:hystrix-helloWorldGroup-5
RUNNABLE --> 5
FALLBACK --> !
RUNNABLE --> 6
Hello 6,current thread:hystrix-helloWorldGroup-7
RUNNABLE --> 7
FALLBACK --> !
RUNNABLE --> 8
Hello 8,current thread:hystrix-helloWorldGroup-9
RUNNABLE --> 9
FALLBACK --> !
RUNNABLE --> 10
FALLBACK --> !
FALLBACK --> !
...
```

### （4）请求结果缓存（Request Cahce）
hystrix支持将一个请求结果缓存起来，下一个具有相同key的请求将直接从缓存中取出结果，减少请求开销。要使用hystrix cache功能，第一个要求是重写`getCacheKey()`，用来构造cache key；第二个要求是构建context，如果请求B要用到请求A的结果缓存，A和B必须同处一个context。通过`HystrixRequestContext.initializeContext()`和`context.shutdown()`可以构建一个context，这两条语句间的所有请求都处于同一个context，同一个context中可以从缓存中直接获取cache key相同的响应结果。

### 例子4（请求结果缓存）：
```
import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.strategy.concurrency.HystrixRequestContext;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/9/28 15:32
 */
public class HelloWorldRequestCacheCommand extends HystrixCommand<Boolean> {

	private final Integer value;
	private final String name;

	public HelloWorldRequestCacheCommand(Integer value,String name) {
		//最小配置,指定groupKey
		super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("helloWorldGroup"))
				//commonKey
				.andCommandKey(HystrixCommandKey.Factory.asKey("helloWorldRequestCache")));
		this.value = value;
		this.name  = name;
	}

	@Override
	protected Boolean run() throws Exception {
		return value == 0 || value %2 == 0;
	}

	@Override
	protected String getCacheKey() {
		return name + value;
	}

	public static void main(String[] args) throws Exception {
		HystrixRequestContext context = HystrixRequestContext.initializeContext();
		try {
			HelloWorldRequestCacheCommand command1 = new HelloWorldRequestCacheCommand(1,"doge");
			HelloWorldRequestCacheCommand command2 = new HelloWorldRequestCacheCommand(1,"doge");
			HelloWorldRequestCacheCommand command3 = new HelloWorldRequestCacheCommand(1,"doge-ex");
			System.out.println("command1 result --> " + command1.execute());
			System.out.println("command1 isResponseFromCache --> " + command1.isResponseFromCache());

			System.out.println("command2 result --> " + command2.execute());
			System.out.println("command2 isResponseFromCache --> " + command2.isResponseFromCache());

			System.out.println("command3 result --> " + command3.execute());
			System.out.println("command3 isResponseFromCache --> " + command3.isResponseFromCache());
		}finally {
			context.shutdown();
		}
		
		Thread.sleep(Integer.MAX_VALUE);
	}
}
```
### 控制台输出：
```
command1 result --> false
command1 isResponseFromCache --> false
command2 result --> false
command2 isResponseFromCache --> true
command3 result --> false
command3 isResponseFromCache --> false
```

### （5）请求合并（Request Collapsing）
hystrix支持N个请求自动合并为一个请求，这个功能在有网络交互的场景下尤其有用，比如每个请求都要网络访问远程资源，如果把请求合并为一个，将使多次网络交互变成一次，极大节省开销。重要一点，两个请求能自动合并的前提是两者足够“近”，即两者启动执行的间隔（timerDelayInMilliseconds）时长要足够小，默认为10ms，即超过10ms将不自动合并。请求合并需要继承`HystrixCollapser<BatchReturnType, ResponseType, RequestArgumentType>`，三个泛型参数的含义分别是：
- BatchReturnType：createCommand()方法创建批量命令的返回值的类型。 
- ResponseType：单个请求返回的类型。 
- RequestArgumentType：getRequestArgument()方法请求参数的类型。

继承HystrixCollapser后需要覆写三个方法：`getRequestArgument()`、`createCommand()`、`mapResponseToRequests()`。

### 例子5（请求合并）：
```
import com.netflix.hystrix.*;
import com.netflix.hystrix.strategy.concurrency.HystrixRequestContext;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/9/28 15:32
 */
public class HelloWorldRequestCollapsingCommand extends HystrixCollapser<List<Boolean>, Boolean, Integer> {

	private final Integer value;

	public HelloWorldRequestCollapsingCommand(Integer value) {
		this.value = value;
	}

	@Override
	public Integer getRequestArgument() {
		return value;
	}

	@Override
	protected HystrixCommand<List<Boolean>> createCommand(Collection<CollapsedRequest<Boolean, Integer>> collapsedRequests) {
		return new BatchCommand(collapsedRequests);
	}

	private static final class BatchCommand extends HystrixCommand<List<Boolean>> {

		private final Collection<CollapsedRequest<Boolean, Integer>> requests;

		private BatchCommand(Collection<CollapsedRequest<Boolean, Integer>> requests) {
			super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("helloWorldGroup"))
					.andCommandKey(HystrixCommandKey.Factory.asKey("helloWorldRequestCollapsing")));
			this.requests = requests;
		}

		@Override
		protected List<Boolean> run() {
			List<Boolean> response = new ArrayList<>();
			for (CollapsedRequest<Boolean, Integer> request : requests) {
				Integer argument = request.getArgument();
				response.add(0 == argument || argument % 2 == 0);  //这里就是执行单元的逻辑
			}
			return response;
		}
	}

	@Override
	protected void mapResponseToRequests(List<Boolean> batchResponse, Collection<CollapsedRequest<Boolean, Integer>> collapsedRequests) {
		int count = 0;
		for (CollapsedRequest<Boolean, Integer> request : collapsedRequests) {
			request.setResponse(batchResponse.get(count++));
		}
	}


	public static void main(String[] args) throws Exception {
		HystrixRequestContext context = HystrixRequestContext.initializeContext();
		try {
			Future<Boolean> command1 = new HelloWorldRequestCollapsingCommand(1).queue();
			Future<Boolean> command2 = new HelloWorldRequestCollapsingCommand(2).queue();
			Future<Boolean> command3 = new HelloWorldRequestCollapsingCommand(3).queue();
			Future<Boolean> command4 = new HelloWorldRequestCollapsingCommand(4).queue();
			Future<Boolean> command5 = new HelloWorldRequestCollapsingCommand(5).queue();
			//故意sleep超过10ms,第六个命令不会合并到本次批量请求
			TimeUnit.MILLISECONDS.sleep(13);
			Future<Boolean> command6 = new HelloWorldRequestCollapsingCommand(6).queue();

			System.out.println(command1.get());
			System.out.println(command2.get());
			System.out.println(command3.get());
			System.out.println(command4.get());
			System.out.println(command5.get());
			System.out.println(command6.get());
			// note：numExecuted表示共有几个命令执行，1个批量多命令请求算一个，这个实际值可能比代码写的要多，
			// 因为due to non-determinism of scheduler since this example uses the real timer
			int numExecuted = HystrixRequestLog.getCurrentRequest().getAllExecutedCommands().size();
			System.out.println("num executed: " + numExecuted);
			int numLogs = 0;
			for (HystrixInvokableInfo<?> command : HystrixRequestLog.getCurrentRequest().getAllExecutedCommands()) {
				numLogs++;
				System.out.println(command.getCommandKey().name() + " => command.getExecutionEvents(): " + command.getExecutionEvents());
			}
			System.out.println("num logs:" + numLogs);
		} finally {
			context.shutdown();
		}

		Thread.sleep(Integer.MAX_VALUE);
	}
}
```
### 控制台输出：
```
false
true
false
true
false
true
num executed: 2
helloWorldRequestCollapsing => command.getExecutionEvents(): [SUCCESS, COLLAPSED]
helloWorldRequestCollapsing => command.getExecutionEvents(): [SUCCESS, COLLAPSED]
num logs:2
```
## 通过注解使用Hystrix
如果想要通过注解使用Hystrix，需要引入一个第三方依赖`hystrix-javanica`，注解使用方式和编程式大致相同，只是属性参数配置都注解化了。三个核心注解分别为@HystrixCommand、@HystrixProperty和@HystrixCollapser。当然还有和请求缓存相关的三个注解@CacheResult、@CacheRemove、@CacheKey。

### 例子6（注解同步执行）：
```
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;

public class HelloWorldHystrixAnnotation {

	@HystrixCommand(groupKey = "helloWorldHystrixAnnotation",
			commandKey = "helloWorldHystrixAnnotationSync", fallbackMethod = "fallbck")
	public Boolean run(Integer value) {
		return 0 == value || value % 2 == 0;
	}

	public Boolean fallbck() {
		return false;
	}
}
```

### 例子7（注解异步执行）：
```
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.command.AsyncResult;

import java.util.concurrent.Future;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/10/8 17:41
 */
public class HelloWorldHystrixAnnotationAsync {

	@HystrixCommand(groupKey = "helloWorldHystrixAnnotation",
			commandKey = "helloWorldHystrixAnnotationAsync", fallbackMethod = "fallbck")
	public Future<Boolean> run(Integer value) {
		return new AsyncResult<Boolean>() {
			@Override
			public Boolean invoke() {
				return 0 == value || value % 2 == 0;
			}
		};
	}

	public Boolean fallbck() {
		return false;
	}
}
```

### 例子8（注解订阅执行）：
```
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import rx.Observable;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/10/8 17:41
 */
public class HelloWorldHystrixAnnotationObervable {

	@HystrixCommand(groupKey = "helloWorldHystrixAnnotation",
			commandKey = "helloWorldHystrixAnnotationObervable", fallbackMethod = "fallbck")
	public Observable<Boolean> run(Integer value) {
		return Observable.create(subscriber -> {
			try {
				if (!subscriber.isUnsubscribed()) {
					subscriber.onNext(value == 0 || value % 2 == 0);
					subscriber.onCompleted();
				}
			} catch (Exception e) {
				subscriber.onError(e);
			}
		});
	}

	public Boolean fallbck() {
		return false;
	}
}
```

# 总结
Hystrix的一些使用和属性配置介绍到此结束，一些细节的用法没有给出具体的例子，例如注解中的一些参数配置，注解中的请求缓存使用，注解中的请求合并等。有些是个人认为不常用的，或者说，从Hystrix的Github中的WIKI可以得到更加详细的介绍，本文仅仅对最为常用的部分做了总结，如果未能帮助到你，请见谅。另外，如果使用了Spring(Boot)项目在使用`hystrix-javanica`必须把相关的切面类注册到Spring容器，否则你做的一切操作都不会生效。

```
End on 2017-10-8 17:56.
Help yourselves!
我是throwable,在广州奋斗,白天上班,晚上和双休不定时加班,晚上有空坚持写下博客。
希望我的文章能够给你带来收获,共勉。
```

[本文原始链接](https://github.com/zjcscut/Reading-Notes-Repository/blob/master/%E5%85%B6%E4%BB%96/Hystrix.md)
