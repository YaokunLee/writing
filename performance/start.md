# 安卓冷启动优化 - 异步初始化启动器

源码：https://github.com/AILanguageQuizCard/LanguageAILearning/tree/lyk/start_opt 只需要看到start_opt库里的内容即可

建议跟着源码来理解下面的内容

## **前置知识**

IdleHandler

拓扑排序

## **组件设计**

### **效果**

将10次平均启动时间，从825ms优化到了785ms

### **术语解释**

启动时间：从application的attachBaseContext方法作为计时起点，到首屏RecyclerView首个item被渲染出来作为计时终点

### **分析**

对于在启动时需要初始化的任务，它们根据不同的分类标准，可以分为

类别一：是否必须在主线程执行：

1. 一定要在主线程执行的
2. 可在子线程执行的

类别二：是否依赖其他任务：

1. 依赖其他任务的
2. 不依赖其他任务的

类别三：必须在进入首屏前完成与否：

1. 必须在首屏前优化完
2. 可以在首屏后空闲时间优化

我们的框架应该能满足以上任意类别，针对以上情况都可以自如处理。

对于类别一，我们用两个类来区分，MainTask是必须在主线程执行的，Task则是可以在子线程执行的任务；

对于类别二，我们在Task A中将使用一个List来存储它所依赖的其他任务，当这个列表中的某个任务Task B完成，Task A将会收到一个通知，Task A会调用 CountDownLatch 的countDown，直到所有的依赖任务都执行完成，Task A才会从CountDownLatch.await（）离开，开始执行自己的run 方法。

任务之间的依赖关系可能比较复杂，我们会使用拓扑排序来获取执行顺序，并按照这个顺序来执行对应的任务。

对于类别三，我们写了一个DelayInitDispatcher的调度器，直接将空闲任务放到DelayInitDispatcher中执行即可

### **我们是怎么实现类别一 类别二的处理的？**

使用如下：

在Application的onCreate方法中，初始化TaskDispatcher并加入所有task

```Java
    @Override
    public void onCreate() {
        super.onCreate();

        LaunchTimer.startRecord();
        TaskDispatcher.init(XLApplication.this);
        TaskDispatcher dispatcher = TaskDispatcher.createInstance();
        dispatcher.addTask(new InitMMKVTask())
                .addTask(new InitRemoteConfigTask())
                .addTask(new InitFirebaseTask())
                .addTask(new InitFakeTask())
                .addTask(new InitFakeTask2())
                .addTask(new InitFakeTask3())
                .addTask(new InitFakeTask4())
                .addTask(new InitFakeTask5())
                .start();

        dispatcher.await();
        LaunchTimer.endRecord("TaskDispatcher");

    }


// 一个Task的例子
public class InitFakeTask2 extends Task {
    @Override
    public void run() throws Exception {
        Thread.sleep(20);
    }

    @Override
    public List<Class<? extends Task>> dependsOn() {
        ArrayList arrayList = new ArrayList();
        arrayList.add(InitFakeTask.class);
        return arrayList;
    }
}
```

直接看到start方法：

```Java
    @UiThread
    public void start() {
        mStartTime = System.currentTimeMillis();
        if (Looper.getMainLooper() != Looper.myLooper()) {
            throw new RuntimeException("must be called from UiThread");
        }
        if (mAllTasks.size() > 0) {
            mAnalyseCount.getAndIncrement();
            printDependedMsg();
            mAllTasks = TaskSortUtil.getSortResult(mAllTasks, mClsAllTasks);
            mCountDownLatch = new CountDownLatch(mNeedWaitCount.get());

            sendAndExecuteAsyncTasks();

            DispatcherLog.i("task analyse cost " + (System.currentTimeMillis() - mStartTime) + "  begin main ");
            executeTaskMain(); // 注释1
        }
        DispatcherLog.i("task analyse cost startTime cost " + (System.currentTimeMillis() - mStartTime));
    }
```

TaskSortUtil.getSortResult 会返回拓扑排序的结果，sendAndExecuteAsyncTasks方法中，将会按照拓扑排序的结果来执行任务

看到sendAndExecuteAsyncTasks

```Java
private void sendAndExecuteAsyncTasks() {
    for (Task task : mAllTasks) {
        if (task.onlyInMainProcess() && !sIsMainProcess) {
            markTaskDone(task);
        } else {
            sendTaskReal(task);
        }
        task.setSend(true);
    }
}
```

看到sendTaskReal

如果是在主线程中执行的任务，我们会直接放到主线程待执行列表中，在上面代码中的注释1，将会开始执行这时放入的任务，并通过执行完后，调用这里的TaskCallBack函数，会通知依赖该任务的后续任务，后续任务就会调用CountDownLatch 的countdown;

如果是在子线程中执行的任务，就直接放入线程池中执行，runOn()返回的是我们自定义的线程池

```Java
private void sendTaskReal(final Task task) {
    if (task.runOnMainThread()) {
        mMainThreadTasks.add(task);

        if (task.needCall()) {
            task.setTaskCallBack(new TaskCallBack() {
                @Override
                public void call() {
                    TaskStat.markTaskDone();
                    task.setFinished(true);
                    satisfyChildren(task);
                    markTaskDone(task);
                    DispatcherLog.i(task.getClass().getSimpleName() + " finish");

                    Log.i("testLog", "call");
                }
            });
        }
    } else {
        // 直接发，是否执行取决于具体线程池
        Future future = task.runOn().submit(new DispatchRunnable(task,this)); // 注释2
        mFutures.add(future);
    }
}
```

再看到注释2，这里是我们自定义的DispatchRunnable，它继承自Runnable，看到下面它的run方法，它会先调用任务的waitToSatisfy()，也就是注释3，里面调用的是Task内部的CountDownLatch 的await方法，于是该线程会一直等待，直到它依赖的任务完成，会调用CountDownLatch 的countdown。当所有任务都完成时，会进入try代码块，执行run方法。执行完后，会satisfyChildren唤醒后续任务。

```Java
public class DispatchRunnable implements Runnable {
    private Task mTask;
    private TaskDispatcher mTaskDispatcher;

    public DispatchRunnable(Task task) {
        this.mTask = task;
    }
    public DispatchRunnable(Task task,TaskDispatcher dispatcher) {
        this.mTask = task;
        this.mTaskDispatcher = dispatcher;
    }

    @Override
    public void run() {
        TraceCompat.beginSection(mTask.getClass().getSimpleName());
        DispatcherLog.i(mTask.getClass().getSimpleName()
                + " begin run" + "  Situation  " + TaskStat.getCurrentSituation());

        Process.setThreadPriority(mTask.priority());

        long startTime = System.currentTimeMillis();

        mTask.setWaiting(true);
        mTask.waitToSatisfy(); // 注释3

        long waitTime = System.currentTimeMillis() - startTime;
        startTime = System.currentTimeMillis();

        // 执行Task
        mTask.setRunning(true);
        try {
            mTask.run();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }

        // 执行Task的尾部任务
        // 为什么要设计一个尾部任务？
        Runnable tailRunnable = mTask.getTailRunnable();
        if (tailRunnable != null) {
            tailRunnable.run();
        }

        if (!mTask.needCall() || !mTask.runOnMainThread()) {
            printTaskLog(startTime, waitTime);

            TaskStat.markTaskDone();
            mTask.setFinished(true);
            if(mTaskDispatcher != null){
                mTaskDispatcher.satisfyChildren(mTask);
                mTaskDispatcher.markTaskDone(mTask);
            }
            DispatcherLog.i(mTask.getClass().getSimpleName() + " finish");
        }
        TraceCompat.endSection();
    }

    /**
     * 打印出来Task执行的日志
     *
     * @param startTime
     * @param waitTime
     */
    private void printTaskLog(long startTime, long waitTime) {
        // ...
    }

}
```

## 怎么对类别三进行处理的？

我们实现了一个DelayInitDispatcher 

```Java
public class DelayInitDispatcher {

    private Queue<Task> mDelayTasks = new LinkedList<>();

    private MessageQueue.IdleHandler mIdleHandler = new MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            if(mDelayTasks.size()>0){
                Task task = mDelayTasks.poll();
                new DispatchRunnable(task).run();
            }
            return !mDelayTasks.isEmpty();
        }
    };

    public DelayInitDispatcher addTask(Task task){
        mDelayTasks.add(task);
        return this;
    }

    public void start(){
        Looper.myQueue().addIdleHandler(mIdleHandler);
    }
}
```

\>>>> 插入

如果当 执行注册的 IdleHandler时，突然有用户响应事件进入，需要主线程响应，怎么办？

当执行注册的 `IdleHandler` 时，如果突然有用户响应事件或其他消息进入 `MessageQueue`，需要主线程响应，Android 的消息循环机制会确保这些事件得到及时处理。这是通过 `MessageQueue` 和 `Looper` 的设计来保证的：

1. 事件和消息的优先级：`MessageQueue` 中的消息和事件都有自己的执行时间戳。当新的事件或消息被投递到队列时，它们会根据时间戳（如果有的话）或者入队顺序被安排执行。通常，用户的交互事件，如触摸、点击等，会立即被投递到队列中并具有高优先级。
2. 中断 IdleHandler 执行：`IdleHandler` 的执行发生在消息队列空闲时。如果在执行 `IdleHandler` 的 `queueIdle()` 方法过程中，新的消息或事件被加入到 `MessageQueue`，这个新加入的消息或事件将会打断 `IdleHandler` 的执行流程。`Looper` 会优先处理这些新的消息或事件，确保UI响应的流畅性和及时性。
3. 继续执行 IdleHandler：一旦消息队列中的新消息或事件被处理完毕，如果主线程再次变为空闲状态，`MessageQueue` 会继续检查并执行剩余的或新注册的 `IdleHandler`。这意味着 `IdleHandler` 的执行不会阻塞新消息或事件的处理。

这种设计确保了主线程能够及时响应用户的交互事件，同时也利用空闲时间执行额外的任务，如资源清理或数据预加载，而不会影响到用户界面的响应性。`IdleHandler` 提供了一种平衡主线程工作和利用空闲时间的机制，从而优化应用的性能和用户体验。
