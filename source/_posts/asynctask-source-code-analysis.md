---
title: 安卓AsyncTask源代码剖析
date: 2016-06-03 18:50:51
tags: [安卓, Java] 
---

AsyncTask这个类相信每个安卓开发者都使用过，因为我们经常需要用它在后台完成一些不适合在UI线程中执行的任务。这个类的使用非常简单，它的源代码算上超过一半篇幅的javadoc也只有不到700行，但如果使用不当还是会造成一些不易发现和调试的性能问题。其中最值得注意的问题是多个AsyncTask实例所封装的任务在后台的执行是顺序还是平行的？可别让AsyncTask这个类的名字骗了你，这里的Async只是指任务相对于UI线程来说是异步的，但（默认情况下）所有任务实际上是在同一个后台进程中以顺序的方式执行的。其实通过官方文档可以发现AsyncTask的开发者们对于这个问题也是非常纠结。
<!-- more -->
> Order of execution  
When first introduced, AsyncTasks were executed serially on a single background thread. Starting with DONUT, this was changed to a pool of threads allowing multiple tasks to operate in parallel. Starting with HONEYCOMB, tasks are executed on a single thread to avoid common application errors caused by parallel execution.  
If you truly want parallel execution, you can invoke executeOnExecutor(java.util.concurrent.Executor, Object[]) withTHREAD_POOL_EXECUTOR.

在安卓最初的时候AsyncTasks是顺序执行的，而从1.6开始改成了并行执行，结果从3.0开始为了“避免由并行执行所造成的普遍的程序错误”又改回了最初的顺行执行。
通过本篇文章我想把AsyncTask的源代码扒个干干净净，以后用它的时候可以胸有成竹。

先来看一下AsyncTask的成员变量：


```java
public abstract class AsyncTask<Params, Progress, Result> {
    private static final String LOG_TAG = "AsyncTask";

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

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private static InternalHandler sHandler;

    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;

    private volatile Status mStatus = Status.PENDING;

    private final AtomicBoolean mCancelled = new AtomicBoolean();
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
```
 
   
在这些成员变量中，可以首先关注仅有的两个public的static final成员THREAD_POOL_EXECUTOR和SERIAL_EXECUTOR。它们都是Executor，从名字不难看出它们分别用来并行和顺序执行任务。而36行的sDefaultExecutor的值等于SERIAL_EXECUTOR，正说明AsyncTask默认是顺序执行任务的。再简单介绍下Executor，它是java.util.concurrent包中的一个接口，用来管理一系列Runnable的执行。其代码如下：

第39行的WorkerRunnable的定义如下：


~~~~java
    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
~~~~


可见它就是一个增加了参数的Callable，可它为什么要叫WorkerRunnable呢，叫WorkerCallable不是更好。。。
紧接着有一个FutureTask，它也是java.util.concurrent包中的一个对接口RunnableFuture的具体实现，而RunnableFuture顾名思义是一个实现了Runnable和Future接口的接口，那Future接口是什么东西呢？它是对一个异步运算的结果的抽象，我们可以通过调用它的各种方法来查看运算是否完成，等待运算结束并获得结果等操作。这里正如官方文档中描述的那样，FutureTask用来包装一个Runnable或Callable（这里是WorkerCallable）并提交给Executor来执行。

接下来是之前看到的SerialExecutor的实现，非常容易理解：


~~~~java
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
~~~~


final ArrayDeque<Runnable> mTasks是所有任务的队列，而Runnable mActive是当前正在被执行的任务。被重写的execute()方法new一个Runnable并将其放入mTasks中，在这个Runnable的run()方法中执行完参数中的Runnable的run()后调用scheduleNext()确保队列中下一个任务的执行。这里顺便提一下这里用了ArrayDeque的offer()来将新元素放入队列末端，它与同样定义在java.util.Queue接口的add()的区别在于对于某些对容纳的元素个数有限的Queue的实现类，如果容量已经饱和不能再加入新元素，offer()将返回false而add()则会抛出IllegalStateException。

看完了简单的SerialExecutor，再回过头来看一下更复杂的ThreadPoolExecutor。它的作用是利用一个线程池来执行任务，而这个线程池是通过构造函数中传进入的各项参数配置的。我们平时很少会直接用它，因为有Executors.newCachedThreadPool()之类的更方便使用的ExecutorService，无需配置参数。先看下代码中调用的构造函数的签名：
~~~~java
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory)
~~~~
workQueue是缓存任务的队列，其类型是BlockingQueue，它也是一个java并发库中的参数，它的主要功能是提供了一些获取和加入元素的阻塞方法，比如假设当前队列的容量已达上限，那么调用put(e)试图加入元素的线程将被阻塞，直到有其它线程拿走了队列中的某些元素。它可以很方便地用来实现producer-consumer队列。在AsyncTask中producer是创建任务的UI线程，而consumer即是消化任务的后台线程。corePoolSize和maximumPoolSize是线程池大小的两个边界。当一个新任务通过execute()被提交的时候，如果当前运行线程数少于corePoolSize，那么将会创建一个新线程来执行任务，即使有些线程是空闲的。如果正在运行的线程数介于这两个值之间，那么只有在workQueue已饱和的时候才会创建新线程，否则任务将会进入workQueue。keepAliveTime的作用是如果线程池中线程个数超过了corePoolSize，那么处于空间状态时间超过keepAliveTime的线程将被终止。TimeUnit unit即keepAliveTime的单位。ThreadFactory是一个简单的接口，它唯一的成员方法是Thread newThread(Runnable r)，即基于一个Runnable生成一个线程。所以这一系列的参数可以让我们细致地对线程池进行配置优化。大家自己观察下被传入的实参分别是什么。

接来下的getHandler()静态函数使用单例模式获得一个InternalHandler，它的作用是在UI线程中执行updateProgress以及doInBackground()结束后的任务，它的定义如下：


~~~~java
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
~~~~


代码很好理解，如果Message是MESSAGE_POST_RESULT，表明此时doInBackground()已完成，调用AsyncTask的finish()方法;如果Messsage是MESSAGE_POST_PROGRESS，则回调AsyncTask的可被用户重写的onProgressUpdate()方法响应进度更新事件。finish()方法的定义如下：


~~~~java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
~~~~


首先判断AsyncTask是否被用户通过调用cancel()方法给取消了，如果是则回调onCancelled()，否则回调onPostExecute()。最后将AsyncTask的状态设为Status.FINISHED。注意到这里两个静态嵌套类InternalHandler和AsyncTaskResult都无可奈何地使用了泛型的AsyncTask的原生类型（RawUseOfParameterizedType），因为AsyncTaskResult中的mData[]既可能是在onProgressUpdate()中的进度类型Progress，也有可能是finish()中的结果类型Result，所以必须要使用原生类型来避免类型检查。

现在可以来看AsyncTask的构造函数了：


~~~~java
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override 
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
~~~~


首先是new一个WorkerRunnable，它的call()方法中调用了doInBackround()并获得其返回的Result。之后以这个WorkerRunnable为参数new了一个FutureTask。

现在来看与我们最相关的execute()方法：


~~~~java
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
~~~~


sDefaultExecutor被初始为SerialExecutor，所以当我们调用execute()时默认情况下任务是在同一后台线程中顺序执行的。AsyncTask提供了一个方法setDefaultExecutor(Executor exec)来设置默认的Executor：


~~~~java
    public static void setDefaultExecutor(Executor exec) {
        sDefaultExecutor = exec;
    }
~~~~


下面来看executeOnExecutor的代码：


~~~~java
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
~~~~


首先判断如果当前状态不为PENDING的话抛出IllegalStateException，因为每个任务只能被执行一次。然后依次执行onPreExecute()和exec.execute(mFuture)。注意在mFuture中会调用doInBackground()。

至此代码就分析完了。
