#### Executor两级调度模型

应用程序将应用分为多个任务，每个任务通过Executor将任务分为多个执行线程，通过线程的执行达到任务调度的工作。

下面介绍一些基本的概念：

+ 任务：Runnable或者Callable代表的任务

+ 任务的执行：任务由Executor执行，通过其子类的实现

+ 异步计算的结果获取：使用Future的get方法获取异步执行的结果 

#### 核心实现ThreadPoolExecutor

ThreadPoolExecutor是线程池的实现类，包括的参数与线程池一致，分别是

+ `corePoolSize`: 核心线程数
+ `BlockingQueue`：任务等待队列
+ `maximumPool`： 最大线程数
+ `RejectedExecutionHandler`：拒绝策略

下面介绍其三种实现

##### FixedThreadPool

特点：核心线程数等于最大线程数，等待队列为无限队列，不会触发拒绝策略，keepAliveTime为0，即多余的空闲线程会被立刻终止。

##### SingleThreadExecutor

特点：核心线程池等于最大线程数等于1，等待队列为无解队列，不会触发拒绝策略。保证串行执行。

##### CachedThreadPool

特点：最大线程数设置为最大，等待队列长度为0，如果超出了核心线程池范围之外，则必须要有空闲线程才能运行，否则立即在最大线程池的范围内创建一个线程来提供运行。



#### 定时实现ScheduledThreadPoolExecutor

定时调度的线程池中，会使用一个延迟队列`DelayQueue`来存储任务，任务会将其调度信息放置到任务信息中，这里的延迟队列起始内部是一个优先队列，按照调度时间进行排序，相同时间调度按照FIFO策略进行调度。线程池从队列头部获取任务并支持即可。

##### 定时调度任务ScheduledFutureTask

定时调度任务主要包含三个重要的时间描述信息

+ time：任务会被执行的具体时间
+ sequenceNumber：序列号，任务提交的顺序
+ period：调度的时间间隔

##### 执行流程

1. 从延迟队列中获取已经到期的定时任务(当前系统时间大于等于任务定义的执行时间)
2. 执行这个到期的定时任务
3. 线程修改这个任务下次调度的时间(上次调度时间+调度时间间隔)
4. 将这个定时任务插入延迟队列(优先队列)，等待下次调度.



#### 异步任务FutureTask

##### 异步任务的执行状态

+ 未执行：未执行`FutureTask.run()`
+ 正在执行:  执行`FutureTask.run()`逻辑
+ 执行完毕：`FutureTask.run()`之后

##### FutureTask的实现

FutureTask基于AQS实现. 对于AQS的基本功能，FutureTask做出了如下实现：

+ `acquire`: FutureTask的get方法，实现了该功能，阻塞调用线程，直到AQS状态运行其继续执行.

  get方法会尝试获取同步状态，如果获取成功则直接返回，否则自旋等待当前线程被唤醒

+ `release`: 改变AQS的状态，允许一个或者多个线程被唤醒，cancel方法实现

