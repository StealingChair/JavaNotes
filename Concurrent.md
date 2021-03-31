Executors

#### 1. 创建线程池有哪几种类型？

线程池创建有七种方式，最核心的是最后一种：

- `newSingleThreadExecutor()`：它的特点在于工作线程数目被限制为 1，操作一个无界的工作队列，所以它保证了所有任务的都是被顺序执行，最多会有一个任务处于活动状态，并且不允许使用者改动线程池实例，因此可以避免其改变线程数目；

  ```JAVA
  // SingThreadExecutor -> DEFAULT QUEUE is LinkedBlockingQueue.class
  
  public static ExecutorService newSingleThreadExecutor() {
      return new FinalizableDelegatedExecutorService
          (new ThreadPoolExecutor(1, 1,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()));
  }
  
  public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
          return new FinalizableDelegatedExecutorService
              (new ThreadPoolExecutor(1, 1,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory));
      }
  ```

- `newCachedThreadPool()`：它是一种用来处理大量短时间工作任务的线程池，具有几个鲜明特点：它会试图缓存线程并重用，当无缓存线程可用时，就会创建新的工作线程；如果线程闲置的时间超过 60 秒，则被终止并移出缓存；长时间闲置时，这种线程池，不会消耗什么资源。其内部使用 `SynchronousQueue` 作为工作队列；

  ```java
  // CachedThreadPool -> DEFAULT QUEUE is SynchronousQueue.class
  
  public static ExecutorService newCachedThreadPool() {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
  }
  
  public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
          return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                        60L, TimeUnit.SECONDS,
                                        new SynchronousQueue<Runnable>(),
                                        threadFactory);
  }
  ```

- `newFixedThreadPool(int nThreads)`：重用指定数目`(nThreads)`的线程，其背后使用的是无界的工作队列，任何时候最多有 `nThreads `个工作线程是活动的。这意味着，如果任务数量超过了活动队列数目，将在工作队列中等待空闲线程出现；如果有工作线程退出，将会有新的工作线程被创建，以补足指定的数目 `nThreads`；

  ```java
  // FixedThreadPool -> DEFAULT QUEUE is LinkedBlockingQueue.class
  
  public static ExecutorService newFixedThreadPool(int nThreads) {
      return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
  }
  
  public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
          return new ThreadPoolExecutor(nThreads, nThreads,
                                        0L, TimeUnit.MILLISECONDS,
                                        new LinkedBlockingQueue<Runnable>(),
                                        threadFactory);
      }
  ```

- `newSingleThreadScheduledExecutor()`：创建单线程池，返回 `ScheduledExecutorService`，可以进行定时或周期性的工作调度；

  ```java
  // use wrapper "DelegatedScheduledExecutorService" to invoke constractor of "ScheduledThreadPoolExecutor"
  
  public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
      return new DelegatedScheduledExecutorService
          (new ScheduledThreadPoolExecutor(1, threadFactory));
  }
  
  
  ```

- `newScheduledThreadPool(int corePoolSize)`：和`newSingleThreadScheduledExecutor()`类似，创建的是个`ScheduledExecutorService`，可以进行定时或周期性的工作调度，区别在于单一工作线程还是多个工作线程；

  ```java
  // Use ScheduledThreadPoolExecutor
  // ScheduledThreadPoolExecutor -> DelayedWorkQueue.class
  
  public class ScheduledThreadPoolExecutor
          extends ThreadPoolExecutor
          implements ScheduledExecutorService {
  
      public ScheduledThreadPoolExecutor(int corePoolSize,
                                         ThreadFactory threadFactory) {
          super(corePoolSize, Integer.MAX_VALUE,
                DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
                new DelayedWorkQueue(), threadFactory);
      }
  }
  ```

  

- `newWorkStealingPool(int parallelism)`：这是一个经常被人忽略的线程池，Java 8 才加入这个创建方法，其内部会构建`ForkJoinPool`，利用Work-Stealing算法，并行地 处理任务，不保证处理顺序；

- `ThreadPoolExecutor()`：是最原始的线程池创建，上面1-3创建方式都是对`ThreadPoolExecutor`的封装。

#### 2. 线程池生成时的参数有哪些？

- `corePoolSize`: 线程池最小保存的线程数，即使这些线程处于空闲
- `maximumPoolSize`: 池中允许的最大线程数
- `keepAliveTime`: 当线程数大于内核数(`corePoolSize`)时，这是多余的空闲线程（非核心线程）将在终止之前等待新任务的最长时间
- `unit`: `keepAliveTime`的单位
- `workQueue`: 用于在执行任务之前保留任务的队列。 此队列将仅保存execute方法提交的Runnable任务。
- `threadFactory`: 执行程序创建新线程时要使用的工厂
- `handler`: 因达到线程边界和队列容量而被阻止执行时使用的处理程序

```Java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
                          // do something...
                          }
```

#### 3. 线程池溢出处理逻辑是什么？

首先查看`corePoolSize` 是否到达阈值，再看`workQueue`是否已满，如果未满放入`workQueue`中；满了则看`maximumPoolSize`是否到达阈值，未到达则扩容线程池，直至到达`maximumPoolSize`阈值，最终调用饱和策略。

#### 4. 饱和策略有哪几种？

1. `AbortPolicy`:直接抛出一个异常，**默认策略**
2. `DiscardPolicy`: 直接丢弃任务
3. `DiscardOldestPolicy`:抛弃下一个将要被执行的任务(**最旧任务**) 
4. `CallerRunsPolicy`:主线程中执行任务





## Queue

#### 1. 常用的Queue有几种？

- 

