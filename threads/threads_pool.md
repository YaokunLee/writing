# 安卓线程池

## Android开发艺术探索》第11章-Android 的线程和线程池读书笔记

https://blog.csdn.net/willway_wang/article/details/122632665

#### 2.1.1 在 Android 中，可以扮演线程角色的类有哪些？

Thread，AsyncTask，HandlerThread 和 IntentService。在 Jetpack 中，新增加的 WorkManager。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NWEzZDM0NTNjNjc0NGQ1NzIyZDEwYjIzODFmNDQ1MTlfR0w1dnVjV2EzMk43TE5mTjZCTldNYlkxZ25MNlY2SUhfVG9rZW46VUJXUmIyMlJOb3NsVjl4VWNaemNYTldYbkJiXzE3MDgxNDY5NzY6MTcwODE1MDU3Nl9WNA)

#### 2.1.2 线程池的好处是什么？

线程的特点：

- 线程是一种受限的系统资源，即线程不可能无限制地产生；
- 线程的创建和销毁都会有相应的开销；
- 线程的调度是系统通过时间片轮转的方式来进行，这也有一定的开销，线程很多时，开销更大。

线程池可以缓存一定数量的线程，避免因为频繁创建和销毁线程带来的系统开销。

- 重用线程池中的线程，避免因为线程的创建和销毁所带来的的性能开销；
- 能够有效地控制线程池的最大并发数，避免大量的线程间因相互抢占系统资源而导致的阻塞现象；
- 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行。

#### 2.2.2 AsyncTask 的两个操作方法，三个泛型参数和五个核心方法分别是什么？

AsyncTask 是一个抽象的泛型类，它提供了 Params、Progress 和 Result 这三个泛型参数。其中 Params 表示异步任务的输入参数的类型，如下载链接等，Progress 表示异步任务的执行进度的类型，比如进度的百分数，Result 表示异步任务的返回结果的类型，如下载文件的长度。如果确实不需要某个具体的参数，可以传入 Void 来代替。

五个核心方法：

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=MTYzYzFmOTk3YzM3ZDNjMWRlNTY1ODQ1OWRlYmFiMThfZHBXUU1NRjVSSlhWbEdNMnkxbmZSNjlvakdUWG1XdTlfVG9rZW46UWVMeGIwcHhUbzJmc1l4cHkwVGNaNVllbjhjXzE3MDgxNDY5NzY6MTcwODE1MDU3Nl9WNA)

#### 2.2.4 AsyncTask 的[类加载](https://so.csdn.net/so/search?q=类加载&spm=1001.2101.3001.7020)是在哪里完成的？会完成哪些工作？

`AsyncTask` 的类加载在 Android4.1及以上已经被系统自动完成，具体是在 `ActivityThread` 的 `main` 方法中，调用 `AsyncTask` 的 `init` 方法来完成的。

```Java
public static void main(String[] args) {
    // 创建主消息循环的 Looper 对象
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    // 初始化 AsyncTask
    AsyncTask.init();
    // 开启主消息循环
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

我们知道，静态成员在加载类的时候会进行初始化，所以我们还应该关注 `AsyncTask` 有哪些静态成员被初始化了。

```Java
public abstract class AsyncTask<Params, Progress, Result> {
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static final InternalHandler sHandler = new InternalHandler();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    public static void init() {
        sHandler.getLooper();
    }
    private static class InternalHandler extends Handler {
    }

}
```

可以看到，有两个线程池（THREAD_POOL_EXECUTOR 和 SERIAL_EXECUTOR）和一个 Handler sHandler 被初始化了。

需要说明的是，sHander 这个对象是在主线程创建的，这样在工作线程中才能通过 sHander 将数据从工作线程切换到主线程中。

\>>>> 插入

Q: 安卓中的SerialExecutor 是什么？

A：SerialExecutor在Android中是一个工具，用于确保提交给它的任务按提交顺序依次执行。它通常用于场景，比如当你需要按顺序执行多个任务，但这些任务又不适合在UI线程（主线程）上直接执行时。SerialExecutor背后的基本思想是，尽管有多个任务被提交给它，但在任何给定时刻，只有一个任务在执行。这对于避免并发问题非常有用，特别是在需要顺序执行任务以保持数据一致性或避免竞争条件的场景中。

SerialExecutor不是Android SDK的官方部分，但它可以作为一个有用的实用程序类在应用中实现或通过搜索在线资源找到。这种执行器特别有用于处理数据库操作、文件读写操作或其他需要顺序执行以避免冲突的任务。通过这种方式，开发者可以在保证应用性能和响应性的同时，确保任务的正确执行顺序。

#### 2.2.5 AsyncTask 的工作原理是什么？ 

自己去根据上面教程链接去看，内容有点多，不复制了

\>>>> 插入

Q：安卓中，一个进程中所有的 AsyncTask 都会在这个串行的线程池中（也就是SerialExecutor ）排队执行，对么？

A： 是的，您的理解基本正确。在Android中，`AsyncTask`类用于在后台线程上执行异步任务，同时提供接口在执行前、执行中和执行后更新UI。在Android 3.0（Honeycomb，API 11）之前，`AsyncTask`任务默认是并行执行的，**但从Android 3.0开始，****`AsyncTask`****的执行策略发生了变化，变为默认在一个串行的线程池中执行，也就是通过一个内部的****`SerialExecutor`****来管理这些任务的执行**。

这意味着从Android 3.0开始，提交给`AsyncTask`的所有任务都会在这个串行的线程池中排队执行。每次只能执行一个任务，一个任务完成后，下一个任务才会开始执行。这个改变是为了避免常见的并发错误，并简化了任务执行的管理。

然而，开发者如果希望并行执行多个`AsyncTask`，也可以通过显式地使用`THREAD_POOL_EXECUTOR`来实现。这样可以让任务并行执行，而不是按照串行的方式排队执行。

```Java
// 串行执行
myAsyncTask.execute();

// 并行执行
myAsyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
```

因此，尽管从Android 3.0开始，`AsyncTask`默认在串行线程池中执行，但开发者仍然有选择并行执行任务的能力。这种设计既提供了简单的方式来避免并发问题，也保留了灵活性以满足那些需要并行处理任务的场景。

#### 2.2.6 AsyncTask 内存泄漏是怎么产生的？如何解决？

当在 Activity 中采用匿名类或者内部类的方式来创建 AsyncTask 时，AsyncTask 默认会持有外部类 Activity 的引用；如果在 Activity 关闭后，AsyncTask 的异步任务还没有执行完毕，那么 AsyncTask 对象就不会被释放，它所持有的 Activity 对象也不会被释放，导致 Activity 对象无法被及时回收，这就会造成内存泄漏。

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NTRmYzljZjg5ZmQxNjQ5MTkxMjNmYzU0MjAwM2E2ZjZfa2wyZFVpOGtjV2Y4U3pHaTVLWHRsc0xJakc3R1NhemtfVG9rZW46TGR3U2JST0d0b2Z5d2t4N3RrdWMyZXY0bmFlXzE3MDgxNDY5NzY6MTcwODE1MDU3Nl9WNA)

解决办法：使用静态内部类来子类化 AsyncTask，如果子类内部要使用到 Activity，采用弱引用的方式来引用 Activity 的实例；在 Activity 结束的时候，及时 cancel 掉 AsyncTask 任务。

#### 2.2.9 IntentService 和 Service 的区别是什么？

联系：IntentService 继承于 Service。

区别：

IntentService 是一个抽象类，包含一个抽象方法 onHandleIntent，因此必须创建它的子类才能创建 IntentService，而 Service 是一个具体类。

IntentService 可以异步执行耗时的任务，当任务执行完毕后它会自动停止；而 Service 是在主线程运行的，不可以直接执行耗时任务，必须自己单独开启工作线程来执行耗时任务。

#### 2.3.2 ThreadPoolExecutor 执行任务的规则是什么？

如果线程池中的线程数量未达到核心线程的数量，那么会直接启动一个核心线程来执行任务；

如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行；

如果在步骤 2 中无法将任务插入到任务队列中（原因往往是任务队列已满），这个时候如果线程数量未达到线程池规定的最大值，那么会启动一个非核心线程来执行任务；

如果步骤 3 中线程数量已经达到线程池规定的最大值，那么就拒绝执行此任务。

绘制流程图如下：

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTM2NWJmNDkyMWFkMTNlYTQxNGYxMWUwOGFjNjk3Y2VfaXlWVGZXUTlYN0ZuUVJPNEFXdWNJa3R1cHVxVUZ1MmhfVG9rZW46VG9TT2IxTU9Zb0F5UkF4VERPSWNtUGNXbklnXzE3MDgxNDY5NzY6MTcwODE1MDU3Nl9WNA)

#### 2.3.3 ThreadPoolExecutor 的 UML 类图是什么？

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=YzI3OWYxODY5OTAwMTk3ZDQ2NzBmMDBkMGE2NjBiN2FfblJDTUxHWEVFcHVVTkt3Z21oODFLeldRMXBTdzRJNzFfVG9rZW46VllGVmJ1Um9IbzhJNDd4d3M0amN2MkpEbkpkXzE3MDgxNDY5NzY6MTcwODE1MDU3Nl9WNA)

顶层接口 Executor 提供了一种思想：**将任务提交和任务执行进行解耦**。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需要提供 Runnable 对象，将任务的运行逻辑提交到执行器（Executor）中，由 Executor 框架完成线程的调度和任务的执行部分。

ExecutorService 接口继承了 Executor 接口，增加了一些能力：

（1）扩充执行任务的能力，补充可以为一个或一批异步任务生成 Future 的方法；

（2）提供了管控线程池的方法，如停止线程池的运行，判断线程池是否在运行。

AbstractExecutorService 是上层的抽象类，提供了对 ExecutorService 接口的默认实现（submit 方法、invokeAll 方法），添加了 newTaskFor 方法来返回一个 RunnableFuture，没有对核心方法 execute 进行实现。它的作用是将执行任务的流程串联了起来，保证下层的实现只需要关注一个执行任务的方法即可。

ThreadPoolExecutor 继承于 AbstractExecutorService 类，是一个具体实现类，它一方面维护自身的生命周期，另一方面同时管理任务和线程，使两者良好地结合从而执行并行任务。

ScheduledExecutorService 接口继承了 ExecutorService 接口，增加可安排在给定的延迟后运行或定期执行的能力。

ScheduledThreadPoolExecutor 继承于 ThreadPoolExecutor 类并实现了 ScheduledExecutorService 接口，在 ThreadPoolExecutor的基础上扩展了可安排在给定的延迟后运行或定期执行的能力。

#### 2.3.4 Executors 里的线程池有哪些？这些线程池有什么特点？

![img](https://is0frj68ok.feishu.cn/space/api/box/stream/download/asynccode/?code=NWY2ZDlhZDVjZmViMmVmYmU2NzA5NDJjYTI5ZTMyYTVfb1dUclZyZGNZYXVQRmEycUZzUUNQU2VZSzFLT2Y5d2tfVG9rZW46WTBFR2I2STQ1bzBnclF4bmtENmNmZ1pSbmllXzE3MDgxNDY5NzY6MTcwODE1MDU3Nl9WNA)


## 拒绝策略

教材来源：https://www.cnblogs.com/jelly12345/p/15006708.html

### **线程池的拒绝策略**

线程池中，有三个重要的参数，决定影响了拒绝策略：corePoolSize - 核心线程数，也即最小的线程数。workQueue - 阻塞队列 。 maximumPoolSize - 最大线程数 当提交任务数大于 corePoolSize 的时候，会优先将任务放到 workQueue 阻塞队列中。当阻塞队列饱和后，会扩充线程池中线程数，直到达到 maximumPoolSize 最大线程数配置。此时，再多余的任务，则会触发线程池的拒绝策略了。总结起来，也就是一句话，当提交的任务数大于（workQueue.size() + maximumPoolSize ），就会触发线程池的拒绝策略。

### **拒绝策略定义**

拒绝策略提供顶级接口 RejectedExecutionHandler ，其中方法 rejectedExecution 即定制具体的拒绝策略的执行逻辑。 jdk默认提供了四种拒绝策略：

- CallerRunsPolicy - 当触发拒绝策略，只要线程池没有关闭的话，则使用调用线程直接运行任务。一般并发比较小，性能要求不高，不允许失败。但是，由于调用者自己运行任务，如果任务提交速度过快，可能导致程序阻塞，性能效率上必然的损失较大
- AbortPolicy - 丢弃任务，并抛出拒绝执行 RejectedExecutionException 异常信息。线程池默认的拒绝策略。必须处理好抛出的异常，否则会打断当前的执行流程，影响后续的任务执行。
- DiscardPolicy - 直接丢弃，其他啥都没有
- DiscardOldestPolicy - 当触发拒绝策略，只要线程池没有关闭的话，丢弃阻塞队列 workQueue 中最老的一个任务，并将新任务加入

### **测试代码**

**1、AbortPolicy**

```Java
    public static void main(String[] args) throws Exception{int corePoolSize = 5;int maximumPoolSize = 10;long keepAliveTime = 5;
        BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<Runnable>(10);
        RejectedExecutionHandler handler = new ThreadPoolExecutor.DiscardOldestPolicy();
        ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.SECONDS, workQueue, handler);for(int i=0; i<100; i++) {try {
                executor.execute(new Thread(() -> log.info(Thread.currentThread().getName() + " is running")));
            } catch (Exception e) {
                log.error(e.getMessage(),e);
            }
        }
        executor.shutdown();
    }
```

executor.execute（）提交任务，由于会抛出 RuntimeException，如果没有try.catch处理异常信息的话，会中断调用者的处理流程，后续任务得不到执行（跑不完100个）。可自行测试下，很容易在控制台console中能查看到。

**2、CallerRunsPolicy** 主体代码同上，更换拒绝策略：

```Java
RejectedExecutionHandler handler = new ThreadPoolExecutor.CallerRunsPolicy();
```

运行后，在控制台console中能够看到的是，会有一部分的数据打印，显示的是 “main is running”，也即体现调用线程处理。

**3、DiscardPolicy** 更换拒绝策略

```Java
RejectedExecutionHandler handler = new ThreadPoolExecutor.DiscardPolicy();
```

直接丢弃任务，实际运行中，打印出的信息不会有100条。

**4、DiscardOldestPolicy** 同样的，更换拒绝策略：

```Java
RejectedExecutionHandler handler = new ThreadPoolExecutor.DiscardOldestPolicy();
```

实际运行，打印出的信息也会少于100条。

**五、总结** 四种拒绝策略是相互独立无关的，选择何种策略去执行，还得结合具体的业务场景。实际工作中，一般直接使用 ExecutorService 的时候，都是使用的默认的 defaultHandler ，也即 AbortPolicy 策略。