### Java线程池研究

#### Java线程状态

1. **NEW （新建状态）**：一个尚未启动的线程所处的状态
2. **RUNNING（可运行状态）**：可运行线程的线程状态，可能正在运行，也可能在等待处理器资源
3. **BLOCAKED（锁阻塞）**：被阻塞等待监视器锁定的线程所处的状态
4. **WAITING（无线等待，阻塞）**：未指定等待时间的线程所处的状态，eg：Object.join()、Object.wait()，通过notify()、notifyAll()来唤醒该线程
5. **TIMED_WAITING（定时等待，阻塞）**：指定等待时间的线程所处的状态
6. **TERMINATED（终止状态）**：已经执行完成的线程状态。

#### 线程池概念

​	创建若干个可执行的线程放入一个池（容器）中，有任务需要处理时，会提交到线程池中的任务队列，处理完之后线程并不会被销毁，而是仍然在线程池（Thread Pool）中等待下一个任务。

​	线程池解决的核心问题是资源管理问题，在并发情况下，系统不能够确定在任意时刻中，有多少任务需要执行，有多少资源需要投入，这种不确定性可能带来如下问题：

1. 频繁申请/销毁资源和调度资源，带来额外的消耗；

2. 对资源无线申请缺少抑制手段，易引发系统资源耗尽的风险；

3. 系统无法合理管理内部资源，会降低系统的稳定性。

   为解决资源分配问题，线程池采取了“池化”（Pooling）思想，为了最大化收益并最小化风险，而将资源统一在一起管理的一种思想。

#### 线程池总体设计

​		线程池首先会创建一些线程，它们的集合成为线程池。使用线程池可以很好地提高性能，线程池在系统启动时即创建大量空闲的线程，程序将一个任务传给线程池，线程池会启动一个线程来执行这个任务，任务执行结束后，这个线程并不会死亡，而是再次返回线程池中成为空闲状态，等待执行下一个任务。它是一种池化技术，使线程可以循环利用，避免了线程的新建和销毁的开销，并有效减少了线程的数量从而也节约了内存资源。

​		Java中的线程池核心实现类是：ThreadPoolExecutor，下图为ThreadPoolExecutor的UML类图。

![image-20210218170241025](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210218170241025.png)

​	**ThreadPoolExecutor**的顶层接口是**Executor**，即用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供**Runnable对象**，将任务的执行逻辑交给**Executor**来完成，而**Executor**框架完成线程的调配和任务的执行，**ExecutorService**接口增加了一些能力：

- 扩充执行任务的能力，补充可以为一个或一批异步任务生成Future的方法

- 提供了管控线程池的方法，比如停止线程池的运行。

  **AbstractExecutorService**是上层的抽象类，保证下层的实现只需关注一个执行任务的方法即可。

  **ThreadPoolExecutor**是最下层的实现部分，**ThreadPoolExecutor**会一方面维护自身的生命周期，另一方面同事管理线程和任务，是两者良好的结合从而执行并行任务。TreadPoolExecutor可以同时维护线程的执行任务，所有任务的调度都是由**execute**方法完成的，这部分完成的工作是：检查线程池的运行状态、运行线程数、运行策略、决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。

#### 线程池生命周期管理

​	线程池内部使用一个变量维护两个值：运行状态（runState）和线程数量（workerCount）。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING,0))
```

ctl它同时包含两部分的信息：线程池的运行状态（runState）和线程池内有效线程的数量（workerCount），高3位保存runState，低29位保存workerCount，两个变量之间互不干扰。用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致而占用锁资源。

​		关于内部封装的获取生命周期状态、获取线程池线程数量的计算方法如下代码：

```
private static int runStateOf(int c) {
// 计算当前运行状态
	return c & ~CAPACITY; 
}

private static int workerCountOf(int c) {
// 计算当前线程数量
	return c & CAPACITY;
}

private static int ctlOf(int rs, int wc) {
// 通过状态和线程数生成ctl
	return rs | wc;
}
```

**ThreadPoolExecutor的运行状态有5种**，分别如下：

| 运行状态   | 状态描述                                                     |
| ---------- | ------------------------------------------------------------ |
| RUNNING    | 能接收新提交的任务，并且也能处理阻塞队列中的任务             |
| SHUTDOWN   | 关闭状态，不再接受新提交的任务，可继续处理阻塞队列中已保存的任务 |
| STOP       | 不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程 |
| TIDYING    | 所有的任务都已终止，workerCount为0                           |
| TERMINATED | 在terminated()方法执行完后进行该状态                         |

#### 线程池任务执行机制

##### 	任务调度

​		所有任务的调度都是由execute方法完成的，其主要工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来要执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝改任务。

![image-20210219100210677](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219100210677.png)

##### 	任务缓冲

​	使用不同的队列可实现不一样的任务存取策略，下表为阻塞队列的成员：

| 名称                  | 描述                                                         |
| --------------------- | ------------------------------------------------------------ |
| ArrayBlockingQueue    | 一个用数组实现的有界阻塞队列，按照FIFO的原则对元素进行排序。支持公平锁和非公平锁。 |
| LinkedBlockingQueue   | 一个由链表结构组成的有界队列，按照FIFO的原则对元素进行排序，队列默认长度为Integer.MAX_VALUE，所以默认创建的该队列有容量危险 |
| PriorityBlockingQueue | 一个支持线程优先级排序的无界队列，默认自然序进行排序，也可以自定义实现compareTo()方法来指定元素排序规则，不能保证同优先级元素的顺序 |
| DelayQueue            | 一个实现PriorityBlockingQueue实现延迟获取的无界队列，在创建元素时，可以指定多久才能从队列中获取当前元素。只有延时期满后才能从队列中获取元素 |
| SynchronousQueue      | 一个不存储元素的阻塞队列，每一个put操作必须等待take操作，否则不能添加元素。支持公平锁和非公平锁。SynchronousQueue的一个使用场景是在线程池中。Executors.newCacheThreadPool()就使用了SynchronousQueue，这个线程池根据需要创建新的线程，如果有空闲线程则会重复使用，线程空闲了60秒后会被回收。 |
| LinkedTransferQueue   | 一个由链表结构组成的无界阻塞队列，相当于其他队列，LinkedTransferQueue队列多了transfer和tryTransfer方法 |
| LinkedBlockingDeque   | 一个由链表结构组成的双向阻塞队列，队列头部和尾部可以添加和移除元素，多线程并发时，可以将锁的竞争最多降到一半。 |

##### 	任务申请

​	任务执行有两种可能：一种是任务直接由新创建的线程执行，一种是线程从任务队列中获取任务然后执行，执行完任务的空闲线程会再次从队列中申请任务再执行。第一种情况仅出现在线程处室创建的时候，第二种是线程获取任务绝大多数的情况。实现线程管理模块和任务管理模块之间的通信这部分策略是由getTask方法实现，其执行流程如下图所示：

![image-20210219103940087](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219103940087.png)

##### 	任务拒绝

​		任务拒绝模块是线程池的保护部分，线程池有一个最大的容量，当线程池的任务缓存队列已满，并且线程池中的线程数目达到maximumPoolSize时，就需要拒绝该任务，采取任务拒绝策略，保护线程池。

```
public interface RejectedExecutionHandler {
	void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

​	用户可通过实现这个接口去定制拒绝策略，也可选择JDK提供的四种已有拒绝策略，其特点如下：

| 名称                | 描述                                                       |
| ------------------- | ---------------------------------------------------------- |
| AbortPolicy         | 丢弃任务并抛出RejectedExecutionException异常，这是默认策略 |
| DiscardPolicy       | 也是丢弃任务，但是不抛出异常                               |
| DiscardOldestPolicy | 丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）   |
| CallerRunsPolicy    | 由调用者所在的线程处理该任务                               |

#### Worker线程管理

##### Worker线程

线程池为了掌握线程的状态并维护线程的生命周期，设计了线程池内的工作线程Worker。

```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
	final Thread thread; //Worker 持有的线程
	Runnable firstTask;// 初始化的任务，可以为null
}
```

Worker 执行任务的模型如下：

![image-20210219105756360](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219105756360.png)

线程回收过程如下图所示：

![image-20210219110506260](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219110506260.png)

##### Worker线程增加

![image-20210219110637235](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219110637235.png)

##### Worker线程回收

​		线程池中线程的销毁依赖JVM 自动的回收，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM 回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。
​		Worker 被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当Worker 无法获取到任务，也就是获取的任务为空时，循环会结束，Worker 会主动消除自身在线程池内的引用。

​		线程回收的工作是在processWorkerExit 方法完成的，线程的销毁流程如下图所示：

![image-20210219111005076](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219111005076.png)



##### Worker 线程执行任务

在Worker 类中的run 方法调用了runWorker 方法来执行任务，runWorker 方法的执行过程如下：

![image-20210219111116522](C:\Users\wangming\AppData\Roaming\Typora\typora-user-images\image-20210219111116522.png)

#### 线程池线程数设计

- CPU密集型任务

  因为CPU 密集型任务使得CPU 使用率很高，若开过多的线程数，会造成CPU 过度切换，所以尽量使用较小的线程池，线程数量一般为CPU 核心数量

- IO密集型任务

  IO 密集型任务CPU 使用率并不高，因此可以让CPU 在等待IO 的时候有其他线程去处理别的任务，充分利用CPU 时间，可以使用稍大的线程池。

  计算公式如下：

  Nthreads=Ncpu * Ucpu *（1+W/C）
  Ncpu=CPU 的数量
  Ucpu= 目标CPU 使用率
  W/C= 等待时间与计算时间的比率

#### 线程池的优势

**直接创建线程的缺点：**

- 线程的创建和销毁开销非常大，每次使用都重新创建线程会增加系统负担，增大响应时间；
- 手动创建线程不方便进行管理，如果没有对线程数量进行控制会导致创建大量线程消耗系统内存等资源(每个线程需要独立分配栈空间，默认是
  1M，具体大小取决于JDK版本和OS)，另外线程过多也会导致频繁切换线程浪费CPU资源；
- 线程创建在不同平台上并不统一，如不同平台和不同OS下创建线程的最大限制不同；

**使用线程池的优势：**

- 降低系统资源消耗：通过池化技术复用线程，降低线程创建和销毁造成的消耗；
- 提高系统响应速度：当有任务到达时，通过复用已存在的线程，无需等待新线程的创建便能立即执行；
- 提高线程的可管理性：可以方便线程并发数的管控
- 提供额外的功扩展能：线程池具备可拓展性，允许开发人员向其中增加更多的功能

#### 线程池实现方法

##### ThreadPoolExecutor

1. 使用ThreadPoolExecutor 建立自定义线程池：

```
public ThreadPoolExecutor(int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,
    ThreadFactory threadFactory,
    RejectedExecutionHandler handler)
{
}
```

2. 提交任务

```
execute()，执行一个任务，没有返回值。
submit()，提交一个线程任务，有返回值。
submit(Callable<T> task) 能获取到它的返回值， 通过future.get() 获取（ 阻塞直到任务执行完）。一般使用FutureTask+Callable 配合使用（IntentService 中有体现）。
submit(Runnable task, T result) 能通过传入的载体result 间接获得线程的返回值。
submit(Runnable task) 则是没有返回值的，就算获取它的返回值也是null。
Future.get 方法会使取结果的线程进入阻塞状态，知道线程执行完成之后，唤醒取结果的线程，然后返回结果。
```

3. 关闭线程池 

   关闭线程池也有两种方式，shutdown() 会阻止新任务提交到线程池，将线程池切换到SHUTDOWN 状态，并调用interruptIdleWorkers 方法请求中断所有空闲的worker，最后调用tryTerminate 尝试结束线程池，在已提交的任务执行完后关闭。shutdownNow() 不仅会阻止新任务提交，还会“粗暴”的关闭线程池内正在执行的任务，并且会移除任务队列中的任务

##### Executors

​	Executors 提供了4 种不同的创建线程池的方法，使用Executors 建立线程池：

```
// FixedThreadPool
public static ExecutorService newFixedThreadPool(int nThreads){
	return new ThreadPoolExecutor(nThreads, nThreads,OL, TimeUnit.MILLISECONDs,new LinkedBlockingQueue<Runnable>();
}
// SingleThreadExecutor
public static Executorservice newSingleThreadExecutor(){
	return new FinalizableDelegatedExecutorService (new ThreadPoolExecutor(1,1,0L,TimeUnit.MILLISECONDS,new 		
    	LinkedBlockingQueue<Runable>();
}
// CachedThreadPool
public static ExecutorService newCachedThreadPool() {
	return new ThreadPoolExecutor(e, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>());
}
// ScheduledThreadPool
public static ScheduledExecutorService newScheduledThreadPool(int corePoolsize){
	return new ScheduledThreadPoolExecutor(corePoolsize);
}
public scheduledThreadPoolExecutor(int corerolsize){
	super(corePoolsize,Integer.MAX_VALUE,0,NANOSECONDs, new DelayedlorkQueue());
}
```

| 方法                 | 使用场景                                                     | 缺陷                              |
| -------------------- | ------------------------------------------------------------ | --------------------------------- |
| CachedThreadPool     | 用来创建一个可以无限扩大的线程池，适用于负载较轻的场景，执行短期异步任务。（可以使得任务快速得到执行，因为任务时间执行短，可以很快结束，也不会造成cpu过度切换） | 最大线程数不限制大小，容易导致OOM |
| FixedThreadPool      | 创建一个固定大小的线程池，因为采用无界的阻塞队列，所以实际线程数量永远不会变化，适用于负载较重的场景，对当前线程数量进行限制（保证线程数可控，不会造成线程过多，导致系统负载更为严重） | 阻塞队列不限制大小，容易导致OOM   |
| SingleThreadExecutor | 创建一个单线程的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行，适用于需要保证顺序执行各个任务 | 阻塞队列不限制大小，容易导致OOM   |
| ScheduledThreadPool  | 创建一个定长线程池，支持定时及周期性任务执行,适用于执行延时或者周期性任务 | 最大线程数不限制大小，容易导致OOM |

几种线程池的参数设置如下表所示：

| 方法                 | corePoolSize | maximumPoolSize   | keepAliveTime | workQueue           |
| -------------------- | ------------ | ----------------- | ------------- | ------------------- |
| CachedThreadPool     | 0            | Integer.MAX_VALUE | 60s           | SynchronousQueue    |
| FixedThreadPool      | nThreads     | nThreads          | 0             | LinkedBlockingQueue |
| SingleThreadExecutor | 1            | 1                 | 0             | LinkedBlockingQueue |
| ScheduledThreadPool  | corePoolSize | Integer.MAX_VALUE | 0             | DelayedWorkQueue    |

##### ForkJoinPool

ForkJoinPool 主要用于实现“分而治之”的算法，特别是分治之后递归调用的函数，例如quick sort等。ForkJoinPool 最适合的是计算**密集型的任务**，如果存在 I/O，线程间同步，sleep() 等会造成线程长时间阻塞的情况时，最好配合使用ManagedBlocke**r。**

**ForkJoinPool 与ThreadPoolExecutor 区别：**

- 线程池中的每个线程对应一个阻塞队列，ThreadPoolExecutor是所有线程共用一个队列。
- 任务队列先进后出（FILO），ThreadPoolExecutor任务队列为先进先出（FIFO）。
- 支持Work Stealing（工作窃取），帮助临近的队列做任务，ThreadPoolExecutor不支持。

##### 使用注意事项

- 线程池和ThreadLocal，在线程池和ThreadLocal配合使用时需要注意的线程池中的线程是会被复用的，所以ThreadLocal中的值会出现不符合预期的情况。
- Executors，虽然JUC很热心的提供了Executors这个工厂类，但是各个Java开发手册并不推荐使用它，例如Executors中的newFixedThreadPool(int n)和newSingleThreadExecutor()均使用了无界队列(BlockingQueue的大小为Integer.MAX_SIZE)，在极端情况下如果负载过大，会在工作队列中堆积大量的任务导致OOM；而newCachedThreadPool()所设置的最大线程数上限为Integer.MAX_SIZE，如果负载过大会导致线程池一直创建新的线程，最终会因为线程句柄耗尽，JVM抛出异常。