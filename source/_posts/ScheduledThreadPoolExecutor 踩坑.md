---
title: ScheduledThreadPoolExecutor 踩坑
categories: [BUG]
---

ScheduledThreadPoolExecutor 工作中大家都经常会用到，可以用来执行定时任务。最近我在开发监控模块的时候用到了这个类，出现了任务执行到一半他就不执行了的问题。下面是我抽离业务之后的代码:

一、首先建立了一个管理类，用于管理线程池，只开放了定时任务执行方法

```java
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.*;

@Slf4j
public class ThreadPoolManager {

    private static final ThreadPoolManager instance = new ThreadPoolManager();

    public static ThreadPoolManager getInstance() {
        return instance;
    }

    private static final long TIMOUT = 2000L;

    private static final int SCHEDULE_CORE_POOL_SIZE = 10;


    private static final int CORE_POOL_SIZE = 10;

    private static final int MAX_POOL_SIZE = 10;

    private static final int KEEP_ALIVE_TIME = 10;

    private ThreadPoolManager() {
    }

    private static final ExecutorService THREAD_POOL = new ThreadPoolExecutor(CORE_POOL_SIZE,
            MAX_POOL_SIZE, KEEP_ALIVE_TIME, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(), task -> {
        Thread thread = new Thread(task);
        thread.setName("normal-thread-" + thread.getId());
        return thread;
    });

    public static final ScheduledThreadPoolExecutor SCHEDULED_EXECUTOR_POOL
            = new ScheduledThreadPoolExecutor(SCHEDULE_CORE_POOL_SIZE, task -> {
        Thread thread = new Thread(task);
        thread.setName("schedule-thread-" + thread.getId());
        return thread;
    }) {
        /**
         * 重写afterExecute方法
         *
         * @param r the runnable that has completed
         * @param t the exception that caused termination, or null if
         * execution completed normally
         */
        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            super.afterExecute(r, t);
            //这个是 Execute 提交的时候
            if (t != null) {
                log.error("afterExecute里面获取到异常信息" + t.getMessage());
            }

            //如果r的实际类型是FutureTask 那么是submit提交的，所以可以在里面get到异常
            if (r instanceof FutureTask) {
                try {
                    Future<?> future = (Future<?>) r;
                    future.get();
                } catch (Exception e) {
                    log.error("future里面取执行异常", e);
                }
            }
        }
    };

    static {
        SCHEDULED_EXECUTOR_POOL.setRemoveOnCancelPolicy(true);
    }

    public ScheduledFuture<?> scheduleAtFix(Runnable task, String taskInfo, long initialDelay, long period, TimeUnit unit) {
        RunnableWrapper taskWrapper = new RunnableWrapper(task, taskInfo);
        taskWrapper.setShowWaitLog(true);
        return SCHEDULED_EXECUTOR_POOL.scheduleAtFixedRate(taskWrapper, initialDelay, period, unit);
    }

    public void execute(Runnable task) {
        THREAD_POOL.execute(task);
    }

}
```



二、为了监控任务，我建立了一个包装类，来包装任务，打印相关指标

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class RunnableWrapper implements Runnable {
    /**
     * 实际要执行的线程任务
     */
    private final Runnable task;
    /**
     * 最后实施时间
     */
    private long lastExectutionTime;
    /**
     * 线程任务被线程池运行的开始时间
     */
    private long startTime;
    /**
     * 线程任务被线程池运行的结束时间
     */
    private long endTime;
    /**
     * 线程信息
     */
    private final String taskInfo;

    /**
     * 显示等待日志
     */
    private boolean showWaitLog;

    /**
     * 执行间隔时间多久，打印日志
     */
    private long durMs = 1000L;

    /**
     * 次
     */
    private long executionTimes = 0;

    /**
     * 当这个任务被创建出来的时候，就会设置他的创建时间
     * 但是接下来有可能这个任务提交到线程池后，会进入线程池的队列排队
     *
     * @param task     任务
     * @param taskInfo 任务信息
     */
    public RunnableWrapper(Runnable task, String taskInfo) {
        this.task = task;
        this.taskInfo = taskInfo;
        this.lastExectutionTime = System.currentTimeMillis();
    }

    public void setShowWaitLog(boolean showWaitLog) {
        this.showWaitLog = showWaitLog;
    }

    public void setDurMs(long durMs) {
        this.durMs = durMs;
    }

    /**
     * 当任务在线程池排队的时候，这个run方法是不会被运行的
     * 但是当任务结束了排队，得到线程池运行机会的时候，这个方法会被调用
     * 此时就可以设置线程任务的开始运行时间
     */
    @Override
    public void run() {
        this.startTime = System.currentTimeMillis();

        // 此处可以通过调用监控系统的API，实现监控指标上报
        // 用线程任务的startTime-createTime，其实就是任务排队时间
        // 这边打印日志输出，也可以输出到监控系统中
        if (showWaitLog) {
            log.info("任务信息: [{}], 任务排队时间: [{}]ms ,执行次数: [{}]", taskInfo, startTime - lastExectutionTime, executionTimes);
        }

        // 接着可以调用包装的实际任务的run方法
        try {
            task.run();
        } catch (Exception e) {
            log.error("run task error", e);
        }

        // 任务运行完毕以后，会设置任务运行结束的时间
        this.endTime = System.currentTimeMillis();
        this.lastExectutionTime = endTime;

        // 此处可以通过调用监控系统的API，实现监控指标上报
        // 用线程任务的endTime - startTime，其实就是任务运行时间
        // 这边打印任务执行时间，也可以输出到监控系统中
        log.info("任务耗时: [{}]ms", endTime - startTime);
        if (endTime - startTime > durMs) {
            log.warn("任务信息: [{}], 任务执行时间: [{}]ms ,执行次数: [{}]", taskInfo, endTime - startTime, executionTimes);
        }

        executionTimes++;
    }
}
```

三、构造主类，模拟任务执行，这里使用了 10 个线程的线程池来定时执行 20 个任务。

```java
public static void main(String[] args) {
        for (int i = 0; i < 20; i++) {
            Runnable job = new Runnable() {
                @Override
                public void run() {
                    // 模拟任务执行...
                    Long v = RandomUtil.randomLong(500, 1000);
                    log.info("正在执行任务...");
                    try {
                        Thread.sleep(v);
                    } catch (InterruptedException e) {
                        log.error("发生异常了", e);
                    }
                    log.info("执行任务完成...");
                }
            };
            ThreadPoolManager.getInstance().scheduleAtFix(job,
                    StrUtil.format("this is job {} ", i),
                    1, 5, TimeUnit.SECONDS);
        }
  // 避免 main 线程执行完毕
        while (true) {
        }

    }
```

执行结果:

```text
21:45:36.094 [schedule-thread-18] INFO RunnableWrapper - 任务信息: [this is job 5 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-15] INFO RunnableWrapper - 任务信息: [this is job 2 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-16] INFO RunnableWrapper - 任务信息: [this is job 3 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-20] INFO RunnableWrapper - 任务信息: [this is job 7 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-14] INFO RunnableWrapper - 任务信息: [this is job 1 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-22] INFO RunnableWrapper - 任务信息: [this is job 9 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-17] INFO RunnableWrapper - 任务信息: [this is job 4 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-13] INFO RunnableWrapper - 任务信息: [this is job 0 ], 任务排队时间: [1009]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-19] INFO RunnableWrapper - 任务信息: [this is job 6 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.094 [schedule-thread-21] INFO RunnableWrapper - 任务信息: [this is job 8 ], 任务排队时间: [1004]ms ,执行次数: [0]
21:45:36.098 [schedule-thread-16] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-13] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-17] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-21] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-14] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-19] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-20] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-22] INFO Main - 正在执行任务...
21:45:36.098 [schedule-thread-18] INFO Main - 正在执行任务...
21:45:36.099 [schedule-thread-15] INFO Main - 正在执行任务...
21:45:36.605 [schedule-thread-13] INFO Main - 执行任务完成...
21:45:36.606 [schedule-thread-13] INFO RunnableWrapper - 任务耗时: [513]ms
21:45:36.620 [schedule-thread-22] INFO Main - 执行任务完成...
21:45:36.620 [schedule-thread-22] INFO RunnableWrapper - 任务耗时: [527]ms
21:45:36.708 [schedule-thread-17] INFO Main - 执行任务完成...
21:45:36.708 [schedule-thread-17] INFO RunnableWrapper - 任务耗时: [615]ms
21:45:36.951 [schedule-thread-15] INFO Main - 执行任务完成...
21:45:36.951 [schedule-thread-15] INFO RunnableWrapper - 任务耗时: [858]ms
21:45:36.978 [schedule-thread-20] INFO Main - 执行任务完成...
21:45:36.978 [schedule-thread-20] INFO RunnableWrapper - 任务耗时: [885]ms
21:45:37.009 [schedule-thread-14] INFO Main - 执行任务完成...
21:45:37.010 [schedule-thread-14] INFO RunnableWrapper - 任务耗时: [917]ms
21:45:37.036 [schedule-thread-18] INFO Main - 执行任务完成...
21:45:37.036 [schedule-thread-18] INFO RunnableWrapper - 任务耗时: [943]ms
21:45:37.048 [schedule-thread-21] INFO Main - 执行任务完成...
21:45:37.049 [schedule-thread-21] INFO RunnableWrapper - 任务耗时: [956]ms
21:45:37.086 [schedule-thread-16] INFO Main - 执行任务完成...
21:45:37.086 [schedule-thread-16] INFO RunnableWrapper - 任务耗时: [993]ms
21:45:37.101 [schedule-thread-19] INFO Main - 执行任务完成...
21:45:37.101 [schedule-thread-19] INFO RunnableWrapper - 任务耗时: [1008]ms
21:45:37.101 [schedule-thread-19] WARN RunnableWrapper - 任务信息: [this is job 6 ], 任务执行时间: [1008]ms ,执行次数: [0]

```

可以发现我的任务甚至都没有提交完毕，日志就不打印了。我这里想可能是任务队列没有任务了

![image-20230601215033231](https://chengdumarkdown.oss-cn-chengdu.aliyuncs.com/blog/image-20230601215033231.png)

结果任务明明都放在任务队列里面了啊，让我们打个断点看看线程在干什么

![image-20230601214823426](https://chengdumarkdown.oss-cn-chengdu.aliyuncs.com/blog/image-20230601214823426.png)

可以发现线程池中的线程全部处于 WAIT 状态

为什么会这样呢，答案其实就在 `futureTask.get()` 这行代码上

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 如果任务已经在执行中了，那么就进入等待
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

```

这个方法会一直等待线程执行完毕,而线程执行完毕是有 state 的状态来判断的,而 ScheduledThreadPoolExecutor 不会更改 state
参考文章：https://juejin.cn/post/6949346078630084645
