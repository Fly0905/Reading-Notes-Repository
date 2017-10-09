[TOC]
# 单机和分布式下延迟任务的实现总结
延迟任务有别于定式任务，定式任务往往是固定周期的，有明确的触发时间。而延迟任务一般没有固定的开始时间，它常常是由一个事件触发的，而在这个事件触发之后的一段时间内触发另一个事件。延迟任务相关的业务场景如下：
* 场景一：物联网系统经常会遇到向终端下发命令，如果命令一段时间没有应答，就需要设置成超时。
* 场景二：订单下单之后30分钟后，如果用户没有付钱，则系统自动取消订单。
* 场景三：某些支付渠道商会采用退避指数的方式异步回调支付结果（如果同步通知失败），分别是x分钟、x^2分钟、x^3分钟后...回调结果。

下面我们来探讨一些方案，其实这些方案没有好坏之分，和系统架构一样，只有最适合。对于单机和数据量较小的情况下，任意一种方案都可行，考虑的是简单明了和开发速度，尽量避免把系统搞复杂了。而对于分布式或数据量较大的情况下，就需要有一些选择，并不是所有的方案都适合了。

## 延迟任务实现方案的分类
下文把延时任务的实现方案分为两类：
* 基于内存实现延时任务的方案。
* 基于介质实现延时任务的方案。

仅仅是本人自己擅自命名，没什么权威。两者的主要区别是：基于内存实现延时任务的运行时数据都放在内存中，如果程序执行过程中出现问题导致内存中的任务丢失（例如服务器宕机），
需要考虑恢复当时正在运行且未执行完毕的延时任务。基于介质实现的延时任务，这里的介质可以是数据库、K-V结构、消息队列等，待执行的任务存放在一定的介质中被持久化，因此任务丢失的
可能性会比较低，当然也必须苛刻地考虑正在执行的任务（尚未执行完）丢失的情况。

## 基于内存实现延时任务的方案
于内存实现延时任务的方案总结下来主要有以下几种：
* JDK延迟队列，主要由java.util.concurrent.DelayQueue实现。
* JDK的ScheduledThreadPoolExecutor，也就是由调度线程池实现（当然在旧版本的JDK中可以使用Timer和TimerTask，但是ScheduledThreadPoolExecutor出现后可以取代Timer和TimerTask）。
* 时间轮。

### JDK延迟队列
Java中的DelayQueue位于java.util.concurrent包下，作为单机实现，它很好的实现了延迟一段时间后触发事件的需求。由于是线程安全的它可以有多个消费者和多个生产者，从而在某些情况下可以提升性能。DelayQueue本质是封装了一个PriorityQueue，使之线程安全，加上Delay功能，也就是说，消费者线程只能在队列中的消息“过期”之后才能返回数据获取到消息，不然只能获取到null。
之所以要用到PriorityQueue，主要是需要排序。也许后插入的消息需要比队列中的其他消息提前触发，那么这个后插入的消息就需要最先被消费者获取，这就需要排序功能。PriorityQueue内部使用最小堆来实现排序队列。队首的，最先被消费者拿到的就是排序系数最小的那个。使用最小堆让队列在数据量较大的时候比较有优势。使用最小堆来实现优先级队列主要是因为最小堆在插入和获取时，时间复杂度相对都比较好，都是O(logN)。
#### 使用DelayQueue
使用DelayQueue的时候，要求DelayQueue中的元素是java.util.concurrent.Delayed的子类，因此，我们需要实现java.util.concurrent.Delayed接口，覆写两个方法`getDelay()`和`compareTo()`，前者用于控制元素剩余延迟时间，后者用于元素排序。
#### DelayQueue使用例子
#### DelayedTask实现了Delayed接口
```
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/10/9 23:45
 */
public class DelayedTask<T extends Runnable> implements Delayed {

	private final T task;
	private final long deadline;

	public DelayedTask(T task, long deadline) {
		this.task = task;
		this.deadline = deadline;
	}

	//返回与此对象相关的剩余延迟时间，以给定的时间单位表示
	@Override
	public long getDelay(TimeUnit unit) {
		return unit.convert(this.deadline - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
	}

	@Override
	public int compareTo(Delayed other) {
		if (other == this) // compare zero ONLY if same object
			return 0;
		if (other instanceof DelayedTask) {
			DelayedTask x = (DelayedTask) other;
			long diff = deadline - x.deadline;
			if (diff < 0)
				return -1;
			else if (diff > 0)
				return 1;
			else
				return 1;
		}
		long d = (getDelay(TimeUnit.MILLISECONDS) - other.getDelay(TimeUnit.MILLISECONDS));
		return (d == 0) ? 0 : ((d < 0) ? -1 : 1);
	}

	public T getTask() {
		return task;
	}

	public long getDeadline() {
		return deadline;
	}

	@Override
	public boolean equals(Object o) {
		if (this == o) return true;
		if (o == null || getClass() != o.getClass()) return false;
		DelayedTask<?> that = (DelayedTask<?>) o;
		if (deadline != that.deadline) return false;
		return task != null ? task.equals(that.task) : that.task == null;
	}

	@Override
	public int hashCode() {
		int result = task != null ? task.hashCode() : 0;
		result = 31 * result + (int) (deadline ^ (deadline >>> 32));
		return result;
	}
}
```
#### TaskQueueDaemonThread用于管理DelayQueue
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.DelayQueue;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

/**
 * @author throwable
 * @version v1.0
 * @description
 * @since 2017/10/9 23:58
 */
public class TaskQueueDaemonThread {

	private static final Logger LOGGER = LoggerFactory.getLogger(TaskQueueDaemonThread.class);

	private static final Executor EXECUTOR = Executors.newFixedThreadPool(20);

	private static final DelayQueue<DelayedTask> QUEUE = new DelayQueue<>();

	private TaskQueueDaemonThread() {
	}

	private static class LazyHolder {
		private static final TaskQueueDaemonThread TASK_QUEUE_DAEMON_THREAD = new TaskQueueDaemonThread();
	}

	public static TaskQueueDaemonThread getInstance() {
		return LazyHolder.TASK_QUEUE_DAEMON_THREAD;
	}

	public void init() {
		Thread daemonThread = new Thread(this::execute);
		daemonThread.setDaemon(true);
		daemonThread.setName("Task Queue Daemon Thread");
		daemonThread.start();
	}

	private void execute() {
		if (LOGGER.isDebugEnabled()) {
			LOGGER.debug("Daemon thread starts...");
		}
		while (true) {
			try {
				DelayedTask task = QUEUE.take();  //一直阻塞
				if (null != task) {
					Runnable runnable = task.getTask();
					if (null != runnable) {
						EXECUTOR.execute(runnable);
						if (LOGGER.isInfoEnabled()) {
							LOGGER.debug("Execute task:{},current timestamp:{}", runnable, System.currentTimeMillis());
						}
					}
				}
			} catch (Exception e) {
				LOGGER.error("Execute task failed", e);
				break;
			}
		}
	}

	public void putTask(long deadline, Runnable task) {
		DelayedTask delayedTask = new DelayedTask<>(task, deadline);
		QUEUE.put(delayedTask);
	}

	public void removeTask(DelayedTask<Runnable> task) {
		QUEUE.remove(task);
	}

	public static void main(String[] args) throws Exception{
		TaskQueueDaemonThread instance = TaskQueueDaemonThread.getInstance();
		instance.init();
		instance.putTask(System.currentTimeMillis() + 5000, () -> System.out.println("延迟5000毫秒执行"));
		instance.putTask(System.currentTimeMillis() + 6000, () -> System.out.println("延迟6000毫秒执行"));

		Thread.sleep(Integer.MAX_VALUE);
	}
}
```
#### 运行TaskQueueDaemonThread中的main函数，控制台输出如下：
```
00:16:13.133 [Task Queue Daemon Thread] DEBUG org.throwable.TaskQueueDaemonThread - Daemon thread starts...
延迟5000毫秒执行
00:16:18.114 [Task Queue Daemon Thread] DEBUG org.throwable.TaskQueueDaemonThread - Execute task:org.throwable.TaskQueueDaemonThread$$Lambda$2/410424423@4334ea6b,current timestamp:1507565778114
00:16:19.116 [Task Queue Daemon Thread] DEBUG org.throwable.TaskQueueDaemonThread - Execute task:org.throwable.TaskQueueDaemonThread$$Lambda$3/1323468230@50bfb93d,current timestamp:1507565779116
延迟6000毫秒执行
```
### ScheduledThreadPoolExecutor

### 时间轮
时间轮是一种非常惊艳的数据结构。其在Linux内核中使用广泛，是Linux内核定时器的实现方法和基础之一。
按使用场景，大致可以分为两种时间轮：原始时间轮和分层时间轮。
分层时间轮是原始时间轮的升级版本，来应对时间“槽”数量比较大的情况，对内存和精度都有很高要求的情况。我们延迟任务的场景一般只需要用到原始时间轮就可以了。
原始时间轮：一个轮子，有8个“槽”，可以代表未来的一个时间。如果以秒为单位，中间的指针每隔一秒钟转动到新的“槽”上面，就好像手表一样。
如果当前指针指在1上面，我有一个任务需要4秒以后执行，那么这个执行的线程回调或者消息将会被放在5上。
那如果需要在20秒之后执行怎么办，由于这个环形结构槽数只到8，如果要20秒，指针需要多转2圈。位置是在2圈之后的5上面（20 % 8 + 1）。这个圈数需要记录在槽中的数据结构里面。
这个数据结构最重要的是两个指针，一个是触发任务的函数指针，另外一个是触发的总第几圈数。时间轮可以用简单的数组或者是环形链表来实现。
相比DelayQueue的数据结构，时间轮在算法复杂度上有一定优势。DelayQueue由于涉及到排序，需要调堆，插入和移除的复杂度是O(lgn)，而时间轮在插入和移除的复杂度都是O(1)。
时间轮比较好的开源实现是Netty的`HashedWheelTimer`。 

## 基于介质实现延时任务的方案