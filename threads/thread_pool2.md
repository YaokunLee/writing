# Android 线程管理之 ThreadPoolExecutor 自定义线程池



自定义线程池支持以下功能：

1. 支持按任务的优先级去执行；

2. 支持线程池暂停.恢复；

3. 异步结果主动回调主线程

   

## **ThreadPoolExecutor 基础知识**

我们自己来自定义一个线程池，今天来学习一下ThreadPoolExecutor，然后结合使用场景定义一个按照线程优先级来执行的任务的线程池。

​    ThreadPoolExecutor线程池用于管理线程任务队列、若干个线程。

##### **1.）ThreadPoolExecutor构造函数**

```Java
ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue workQueue)
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize,long keepAliveTime, TimeUnit unit,BlockingQueue workQueue,RejectedExecutionHandler handler)
ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue workQueue,RejectedExecutionHandler handler) 
ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue workQueue,ThreadFactory threadFactory, RejectedExecutionHandler handler)
```

- 　　corePoolSize： 线程池维护线程的最少数量
- 　　maximumPoolSize：线程池维护线程的最大数量
- 　　keepAliveTime： 线程池维护线程所允许的空闲时间
- 　　unit： 线程池维护线程所允许的空闲时间的单位
- 　　workQueue： 线程池所使用的缓冲队列
- 　　threadFactory：线程池用于创建线程
- 　　handler： 线程池对拒绝任务的处理策略

##### **2.）创建新线程**

   默认使用Executors.defaultThreadFactory()，也可以通过如下方式

```Java
    /**
     * 创建线程工厂
     */
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        @Override
        public Thread newThread(Runnable runnable) {
            return new Thread(runnable, "download#" + mCount.getAndIncrement());
        }
    };
```

##### **3.）线程创建规则**

​     ThreadPoolExecutor对象初始化时，不创建任何执行线程，当有新任务进来时，才会创建执行线程。构造ThreadPoolExecutor对象时，需要配置该对象的核心线程池大小和最大线程池大小

　　1. 当目前执行线程的总数小于核心线程大小时，所有新加入的任务，都在新线程中处理。

　　2. 当目前执行线程的总数大于或等于核心线程时，所有新加入的任务，都放入任务缓存队列中。

　　3. 当目前执行线程的总数大于或等于核心线程，并且缓存队列已满，同时此时线程总数小于线程池的最大大小，那么创建新线程，加入线程池中，协助处理新的任务。

　　4. 当所有线程都在执行，线程池大小已经达到上限，并且缓存队列已满时，就rejectHandler拒绝新的任务。

##### **4.）默认的RejectExecutionHandler拒绝执行策略**

　　1. AbortPolicy 直接丢弃新任务，并抛出RejectedExecutionException通知调用者，任务被丢弃

　　2. CallerRunsPolicy 用调用者的线程，执行新的任务，如果任务执行是有严格次序的，请不要使用此policy

　　3. DiscardPolicy 静默丢弃任务，不通知调用者，在处理网络报文时，可以使用此任务，静默丢弃没有几乎处理的报文

　　4. DiscardOldestPolicy 丢弃最旧的任务，处理网络报文时，可以使用此任务，因为报文处理是有时效的，超过时效的，都必须丢弃

   我们也可以写一些自己的RejectedExecutionHandler，例如拒绝时，直接将线程加入缓存队列，并阻塞调用者，或根据任务的时间戳，丢弃超过限制的任务。

##### **5.）任务队列BlockingQueue** 

​    排队原则

　　1. 如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不进行排队。

　　2. 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。

　　3. 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。

   常见几种BlockingQueue实现

​     1. ArrayBlockingQueue :  有界的数组队列

　　2. LinkedBlockingQueue : 可支持有界/无界的队列，使用链表实现

　　3. PriorityBlockingQueue : 优先队列，可以针对任务排序

　　4. SynchronousQueue : 队列长度为1的队列，和Array有点区别就是：client thread提交到block queue会是一个阻塞过程，直到有一个worker thread连接上来poll task。

##### **6.)线程池执行**

　　execute()方法中，调用了三个私有方法

　　addIfUnderCorePoolSize()：在线程池大小小于核心线程池大小的情况下，扩展线程池

　　addIfUnderMaximumPoolSize()：在线程池大小小于线程池大小上限的情况下，扩展线程池

　　ensureQueuedTaskHandled()：保证在线程池关闭的情况下，新加入队列的线程也能正确处理

##### **7.）线程池关闭**

　　shutdown()：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务

　　shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务

## **实现优先级线程池**

#####     **1.）定义线程优先级枚举**

```Java
/**
 * 线程优先级
 */
public enum Priority {
    HIGH, NORMAL, LOW
}
```

#####  **2.）定义线程任务**

```Java
/**
 * 带有优先级的Runnable类型
 */
/*package*/ class PriorityRunnable implements Runnable {

    public final Priority priority;//任务优先级
    private final Runnable runnable;//任务真正执行者
    /*package*/ long SEQ;//任务唯一标示

    public PriorityRunnable(Priority priority, Runnable runnable) {
        this.priority = priority == null ? Priority.NORMAL : priority;
        this.runnable = runnable;
    }

    @Override
    public final void run() {
        this.runnable.run();
    }
}
```

##### **3.）定义一个PriorityExecutor继承ThreadPoolExecutor**

```Java
public class PriorityExecutor extends ThreadPoolExecutor {

    private static final int CORE_POOL_SIZE = 5;//核心线程池大小
    private static final int MAXIMUM_POOL_SIZE = 256;//最大线程池队列大小
    private static final int KEEP_ALIVE = 1;//保持存活时间，当线程数大于corePoolSize的空闲线程能保持的最大时间。
    private static final AtomicLong SEQ_SEED = new AtomicLong(0);//主要获取添加任务

    /**
     * 创建线程工厂
     */
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        @Override
        public Thread newThread(Runnable runnable) {
            return new Thread(runnable, "download#" + mCount.getAndIncrement());
        }
    };


    /**
     * 线程队列方式 先进先出
     */
    private static final Comparator FIFO = new Comparator() {
        @Override
        public int compare(Runnable lhs, Runnable rhs) {
            if (lhs instanceof PriorityRunnable && rhs instanceof PriorityRunnable) {
                PriorityRunnable lpr = ((PriorityRunnable) lhs);
                PriorityRunnable rpr = ((PriorityRunnable) rhs);
                int result = lpr.priority.ordinal() - rpr.priority.ordinal();
                return result == 0 ? (int) (lpr.SEQ - rpr.SEQ) : result;
            } else {
                return 0;
            }
        }
    };

    /**
     * 线程队列方式 后进先出
     */
    private static final Comparator LIFO = new Comparator() {
        @Override
        public int compare(Runnable lhs, Runnable rhs) {
            if (lhs instanceof PriorityRunnable && rhs instanceof PriorityRunnable) {
                PriorityRunnable lpr = ((PriorityRunnable) lhs);
                PriorityRunnable rpr = ((PriorityRunnable) rhs);
                int result = lpr.priority.ordinal() - rpr.priority.ordinal();
                return result == 0 ? (int) (rpr.SEQ - lpr.SEQ) : result;
            } else {
                return 0;
            }
        }
    };

    /**
     * 默认工作线程数5
     *
     * @param fifo 优先级相同时, 等待队列的是否优先执行先加入的任务.
     */
    public PriorityExecutor(boolean fifo) {
        this(CORE_POOL_SIZE, fifo);
    }

    /**
     * @param poolSize 工作线程数
     * @param fifo     优先级相同时, 等待队列的是否优先执行先加入的任务.
     */
    public PriorityExecutor(int poolSize, boolean fifo) {
        this(poolSize, MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, new PriorityBlockingQueue(MAXIMUM_POOL_SIZE, fifo ? FIFO : LIFO), sThreadFactory);
    }

    public PriorityExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue workQueue, ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
    }

    /**
     * 判断当前线程池是否繁忙
     * @return
     */
    public boolean isBusy() {
        return getActiveCount() >= getCorePoolSize();
    }

    /**
     * 提交任务
     * @param runnable
     */
    @Override
    public void execute(Runnable runnable) {
        if (runnable instanceof PriorityRunnable) {
            ((PriorityRunnable) runnable).SEQ = SEQ_SEED.getAndIncrement();
        }
        super.execute(runnable);
    }
}
```

里面定义了两种线程队列模式： FIFO（先进先出） LIFO（后进先出） 优先级相同的按照提交先后排序

##### **4.）测试程序**

```Java
 ExecutorService executorService = new PriorityExecutor(5, false);
        for (int i = 0; i < 20; i++) {
            PriorityRunnable priorityRunnable = new PriorityRunnable(Priority.NORMAL, new Runnable() {
                @Override
                public void run() {
                    Log.e(TAG, Thread.currentThread().getName()+"优先级正常");
                }
            });
            if (i % 3 == 1) {
                priorityRunnable = new PriorityRunnable(Priority.HIGH, new Runnable() {
                    @Override
                    public void run() {
                        Log.e(TAG, Thread.currentThread().getName()+"优先级高");
                    }
                });
            } else if (i % 5 == 0) {
                priorityRunnable = new PriorityRunnable(Priority.LOW, new Runnable() {
                    @Override
                    public void run() {
                        Log.e(TAG, Thread.currentThread().getName()+"优先级低");
                    }
                });
            }
            executorService.execute(priorityRunnable);
        }
```

运行结果：不难发现优先级高基本上优先执行了 最后执行的基本上优先级比较低

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=M2U4MTAyYzcxYjEzMmQyYjIwZWZmN2U1ZTE5ZGJmM2RfMXNBNWhVRHpZOVFvbm9BQ2g0U0Jua0phVFl0VUJteFRfVG9rZW46U3ZGMWJHWVpXb0RDRm94RG9WZGM5c3hkbkNkXzE3MDg2NzQ0MDM6MTcwODY3ODAwM19WNA)

## 异步结果主动回调主线程

如果需要异步结果主动回调主线程，需要用户自己继承Callback接口，并在完成onBackground（子线程执行） 和 onCompleted （最终主线程执行）方法

```Java
abstract class Callable<T> : Runnable {
    override fun run() {
        mainHandler.post { onPrepare() }

        val t: T? = onBackground()

        //移除所有消息.防止需要执行onCompleted了，onPrepare还没被执行，那就不需要执行了
        mainHandler.removeCallbacksAndMessages(null)
        mainHandler.post { onCompleted(t) }
    }

    open fun onPrepare() {
        //转菊花
    }

    abstract fun onBackground(): T?
    abstract fun onCompleted(t: T?)
}
```