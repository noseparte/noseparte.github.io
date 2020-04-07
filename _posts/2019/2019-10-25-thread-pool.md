---
layout: post
title:  "网络游戏中的网关线程池是如何创建的"
subtitle:  "Java 线程池的正确使用姿势"
date:   2019-10-25 12:36:26 +0800
tags:
      - thread
---

...
# Java 线程池的正确使用姿势

- [ThreadPool 线程池的定义? 如何创建?](#线程池threadPool)
- [配置 ThreadPoolExecutor](#threadPoolExecutor)
- [管理任务队列 BlockingQueue](#blockingQueue)
- [饱和策略 RejectedExecutionHandler](#rejectedExecutionHandler)
- [[不推荐] 使用Executors工厂模式创建线程池](#executors)
- [ExecutorService的生命周期](#executorService)
- [线程工厂 ThreadFactory](#threadFactory)
    - [DefaultThreadFactory](#defaultThreadFactory) 
    - [PrivilegedThreadFactory](#privilegedThreadFactory) 
    - [[推荐] 使用guava的 ThreadFactoryBuilder](#threadFactoryBuilder) 

- [创建线程池的正确姿势](#end)

## 线程池 ThreadPool

> **1. 线程池的定义：**

（摘自[职Q](https://zq-mobile.zhaopin.com/mAnswer/6161026/)）在面向对象编程中，创建和销毁对象是很费时间的，因为创建一个对象要获取内存资源或者其它更多资源。在Java中更是如此，虚拟机将试图跟踪每一个对象，以便能够在对象销毁后进行垃圾回收。所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数，特别是一些很耗资源的对象创建和销毁，这就是"池化资源"技术产生的原因。线程池顾名思义就是事先创建若干个可执行的线程放入一个池（容器）中，需要的时候从池中获取线程不用自行创建，使用完毕不需要销毁线程而是放回池中，从而减少创建和销毁线程对象的开销。

> **2. 如何创建线程池：**

- **使用 ThreadPoolExecutor：**
ThreadPoolExecutor是一个灵活的、稳定的线程池，允许进行定制。
- **使用 Executors：**
Executors中的静态工厂方法之一来创建线程池：
**newSingleThreadExecutor：** 是一个单线程的Executor，它创建单个工作者线程来执行任务，如果这个线程异常结束，会创建另一个线程来替代。newSingleThreadExecutor能确保依照任务在队列中的顺序来串行执行(例如 FIFO、LIFO、优先级)。
**newFixedThreadPool：** 将创建一个固定长度的线程池，每当提交一个任务时就创建一个线程，直到达到线程池的最大数量，这时线程池的规模将不再发生变化(如果某个线程由于发生了未预期的Exception而结束，那么线程池会补充一个新的线程)。
**newCachedThreadPool：** 将创建一个可缓存的线程池，如果线程池的当前规模超过了处理需求时，那么将回收空闲的线程，而当需求增加时，则可以添加新的线程，线程池规模不存在任何限制。
**newScheduledThreadExecutor：** 创建了一个固定长度的线程池，而且以延迟或定时的方式来执行任务，类似于Timer。


## 配置 ThreadPoolExecutor 

```java
public class ThreadPoolExecutor {
    
    // 线程池维护的最小线程数
    private volatile int corePoolSize;
    // 线程池可容纳线程数的最大值
    private volatile int maximumPoolSize;
    // 线程池达到阈值后，新的线程需要等待的时间
    private volatile long keepAliveTime;
    // 以工厂模式创建新的线程
    private volatile ThreadFactory threadFactory;
    // 上下文
    private final AccessControlContext acc;
    // 阻塞队列
    private final BlockingQueue<Runnable> workQueue;
    // 拒绝策略
    private volatile RejectedExecutionHandler handler;
    
    /**
     * ThreadPoolExecutor的核心构造器
     */
    public ThreadPoolExecutor(int corePoolSize, 
                                       int maximumPoolSize, 
                                       long keepAliveTime,
                                       TimeUnit unit,  
                                       BlockingQueue<Runnable> workQueue,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
           if (corePoolSize < 0 ||    
              maximumPoolSize <= 0 ||   
              maximumPoolSize < corePoolSize ||    
              keepAliveTime < 0)    throw new IllegalArgumentException();
           if (workQueue == null || threadFactory == null || handler == null)    
              throw new NullPointerException();
           this.acc = System.getSecurityManager() == null ?     
                    null :        
                    AccessController.getContext();
           this.corePoolSize = corePoolSize;
           this.maximumPoolSize = maximumPoolSize;
           this.workQueue = workQueue;
           this.keepAliveTime = unit.toNanos(keepAliveTime);
           this.threadFactory = threadFactory;
           this.handler = handler;
    }
    
}
```

## 管理任务队列 BlockingQueue

**ThreadPoolExecutor**允许提供一个**BlockingQueue**来保存等待执行的任务。基本的任务排队方法有3种：无界队列、有界队列和同步移交(Synchronous Handoff)。


- **无界队列：** 队列大小无限制，常用的为无界的LinkedBlockingQueue，使用该队列做为阻塞队列时要尤其当心，当任务耗时较长时可能会导致大量新任务在队列中堆积最终导致OOM。
阅读代码发现，Executors.newFixedThreadPool 采用就是 LinkedBlockingQueue，而楼主踩到的就是这个坑，当QPS很高，发送数据很大，大量的任务被添加到这个无界LinkedBlockingQueue 中，导致cpu和内存飙升服务器挂掉。
- **有界队列：** 常用的有两类，
一类是遵循FIFO原则的队列如ArrayBlockingQueue与有界的LinkedBlockingQueue，
另一类是优先级队列如PriorityBlockingQueue。PriorityBlockingQueue中的优先级由任务的Comparator决定。
使用有界队列时队列大小需和线程池大小互相配合，线程池较小有界队列较大时可减少内存消耗，降低cpu使用率和上下文切换，但是可能会限制系统吞吐量。在我们的修复方案中，选择的就是这个类型的队列，虽然会有部分任务被丢失，但是我们线上是排序日志搜集任务，所以对部分对丢失是可以容忍的。
- **同步移交队列：** 如果不希望任务在队列中等待而是希望将任务直接移交给工作线程，可使用SynchronousQueue作为等待队列。SynchronousQueue不是一个真正的队列，而是一种线程之间移交的机制。要将一个元素放入SynchronousQueue中，必须有另一个线程正在等待接收这个元素。只有在使用无界线程池或者有饱和策略时才建议使用该队列。


## 饱和策略 RejectedExecutionHandler

> ThreadPoolExecutor提供如下4种饱和策略:

- **CallerRunsPolicy** 由调用线程（提交任务的线程）处理该任务
- **AbortPolicy** 丢弃任务并直接抛出RejectedExecutionException异常(默认的线程池拒绝策略)
- **DiscardPolicy** 仅丢弃任务并不抛出异常
- **DiscardOldestPolicy** 丢弃队列最前面的任务，然后重新提交被拒绝的任务

自定义饱和策略，只需实现**RejectedExecutionHandler**接口并重写**void rejectedExecution(Runnable r, ThreadPoolExecutor executor)** 方法



```java
public class ThreadPoolExecutor{

    /** 
     *  默认的线程池拒绝策略 AbortPolicy
     */
    private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();

    /* ThreadPoolExecutor提供如下4中拒绝策略: */
    /**
     * 由调用线程（提交任务的线程）处理该任务
     */
   public static class CallerRunsPolicy implements RejectedExecutionHandler {}
 
    /** 
     *  丢弃任务并直接抛出RejectedExecutionException异常
     */
   public static class AbortPolicy implements RejectedExecutionHandler {}
 
   /** 
    * 仅丢弃任务并不抛出异常
    */
   public static class DiscardPolicy implements RejectedExecutionHandler {}
   
   /** 
    * 丢弃队列最前面的任务，然后重新提交被拒绝的任务
    */
   public static class DiscardOldestPolicy implements RejectedExecutionHandler {}

}
```



## Executors(不推荐)

在阿里巴巴Java开发手册中提到，使用Executors创建线程池可能会导致OOM(OutOfMemory ,内存溢出)

![BlockingQueue致使OOM示意图](https://github.com/AwakenCN/Almost-Famous/blob/aedc643ad298225a54aa1af19e877e45f03f0ff9/famous-static/images/blockingQueue.png?raw=true)

## ExecutorService

```java
public interface ExecutorService extends Executor {  
     void shutdown();    
     List<Runnable> shutdownNow();    
     boolean isShutdown();    
     boolean isTerminated();    
     boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;  
     // .......其他用于任务提交的方法   
}    
```

为了解决执行服务的生命周期问题，
**ExecutorService**拓展了Executor接口，添加了一些用于生命周期管理的方法。
**ExecutorService**的生命周期有3种状态：运行、关闭和已终止。
**ExecutorService**在初始创建时处于运行状态。
**shutdown**方法将执行平缓的关闭过程：不再接受新的任务，同时等待已经提交的任务执行完成——包括那些还未开始执行的任务。
**shutdownNow**方法将执行粗暴的关闭过程：它将尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务。

## ThreadFactory 

### DefaultThreadFactory

```java

/** * The default thread factory */
static class DefaultThreadFactory implements ThreadFactory {  

        private static final AtomicInteger poolNumber = new AtomicInteger(1);    
        private final ThreadGroup group;    
        private final AtomicInteger threadNumber = new AtomicInteger(1);    
        private final String namePrefix;    
    
        DefaultThreadFactory() {        
            SecurityManager s = System.getSecurityManager();        
            group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();        
            namePrefix = "pool-" +                      
                              poolNumber.getAndIncrement() +                     
                              "-thread-";    
        }    
    
        public Thread newThread(Runnable r) {       
            Thread t = new Thread(group, r,                              
                            namePrefix + threadNumber.getAndIncrement(),
                            0);
            if (t.isDaemon())            
                t.setDaemon(false);       
            if (t.getPriority() != Thread.NORM_PRIORITY)      
                t.setPriority(Thread.NORM_PRIORITY);       
            return t;   
        }
}

```

### PrivilegedThreadFactory

```java
/**
 * 权限访问与类加载
 */
static class PrivilegedThreadFactory extends DefaultThreadFactory {    

    private final AccessControlContext acc;    
    private final ClassLoader ccl;    
    PrivilegedThreadFactory() {        
        super();        
        SecurityManager sm = System.getSecurityManager();        
        if (sm != null) {            
            // Calls to getContextClassLoader from this class           
            // never trigger a security check, but we check            
            // whether our callers have this permission anyways.            
            sm.checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);            
            // Fail fast            
            sm.checkPermission(new RuntimePermission("setContextClassLoader"));        
        }        
        this.acc = AccessController.getContext();        
        this.ccl = Thread.currentThread().getContextClassLoader();    
    }    
    
    public Thread newThread(final Runnable r) {        
        return super.newThread(new Runnable() {            
            public void run() {                
                AccessController.doPrivileged(new PrivilegedAction<Void>() {                    
                    public Void run() {   
                        Thread.currentThread().setContextClassLoader(ccl);                        
                        r.run();                        
                         return null;                   
                    }                
                }, acc);            
            }        
        });    
    }
}

```


### 使用guava的 ThreadFactoryBuilder

```java

public class ThreadFactoryBuilder{

    private static ThreadFactory doBuild(ThreadFactoryBuilder builder) {  
        final String nameFormat = builder.nameFormat;  
        final Boolean daemon = builder.daemon;  
        final Integer priority = builder.priority;  
        final UncaughtExceptionHandler uncaughtExceptionHandler = builder.uncaughtExceptionHandler;  
        final ThreadFactory backingThreadFactory =      
             (builder.backingThreadFactory != null)          
                ? builder.backingThreadFactory         
                : Executors.defaultThreadFactory();  
        final AtomicLong count = (nameFormat != null) ? new AtomicLong(0) : null;  
        return new ThreadFactory() {    
            @Override    
            public Thread newThread(Runnable runnable) {      
                Thread thread = backingThreadFactory.newThread(runnable);      
                if (nameFormat != null) {        
                    thread.setName(format(nameFormat, count.getAndIncrement()));      
                }     
                if (daemon != null) {// 守护线程        
                    thread.setDaemon(daemon);     
                }     
                if (priority != null) {// 优先级        
                    thread.setPriority(priority);      
                }      
                if (uncaughtExceptionHandler != null) {     
                    thread.setUncaughtExceptionHandler(uncaughtExceptionHandler);     
                }      
                return thread;   
            } 
        };
    }
}

```

## 创建线程池的正确姿势

```java
/** 
 * @Auther: Noseparte * @Date: 2019/11/27 10:35 
 * @Description: 
 * 
 *          <p>定制协议网关线程池</p> 
 */
public class ThreadPool {    

    protected final static Logger _LOG = LogManager.getLogger(ThreadPool.class);    
    private List<ExecutorService> workers = new ArrayList<>();    
    private int threadCount;    
    private ThreadFactory threadFactory;
    
    public ThreadPool(int threadCount) {        
        this(threadCount, new UserThreadFactory("网关游戏逻辑协议线程池"));    
    }    
    
    public ThreadPool(int threadCount, ThreadFactory threadFactory) {        
        this.threadCount = threadCount;        
        this.threadFactory = threadFactory;        
        if (threadCount <= 0 || null == threadFactory)            
            throw new IllegalArgumentException();        
            for (int i = 0; i < threadCount; i++) {            
                workers.add(new ThreadPoolExecutor(threadCount, 200,                    
                    0L, TimeUnit.MILLISECONDS,                    
                    new LinkedBlockingQueue<Runnable>(1024),
                    threadFactory, 
                    new ThreadPoolExecutor.AbortPolicy()));        
            }    
    }    
    
    public Future execute(Runnable task, int mold) {        
        int index = Math.abs(mold) % threadCount;        
        ExecutorService executor = workers.get(index);        
        if (null == executor) {           
            _LOG.error("sid=" + mold + ", tid=" + index);            
            return null;       
        }        
        return executor.submit(task);    
    }    
    
    public void shutdown() {       
        int count = 0;        
        for (ExecutorService worker : workers) {            
            _LOG.error("close thread{}.", ++count);            
            worker.shutdown();        
        }   
    }    
    
    static class UserThreadFactory implements ThreadFactory {        
        private static final AtomicInteger poolNumber = new AtomicInteger(1);        
        private final ThreadGroup group;        
        private final AtomicInteger threadNumber = new AtomicInteger(1);        
        private final String namePrefix;        
        
        UserThreadFactory(String poolName) {            
            SecurityManager s = System.getSecurityManager();            
            group = (s != null) ? s.getThreadGroup() :  
                    Thread.currentThread().getThreadGroup();            
            namePrefix = poolName + "-" +                    
                    poolNumber.getAndIncrement() +                    
                    "-thread-";        
        }     
        
        public Thread newThread(Runnable r) {            
            Thread t = new Thread(group, r,                    
                namePrefix + threadNumber.getAndIncrement(),                    
                0);            
            if (t.isDaemon())                
                t.setDaemon(false);            
            if (t.getPriority() != Thread.NORM_PRIORITY)   
                t.setPriority(Thread.NORM_PRIORITY);            
            return t;        
        }   
        
    }
}
```


## 总结

> **创建线程池的注意事项：**
1. 根据业务场景定制ThreadFactory、饱和策略、任务队列、ThreadPoolExecutor
2. 注意BlockingQueue中任务阻塞数量越来越多会导致内存耗尽(OOM), 要设置队列的上限值

> **源码地址：**
[Almost-Famous: 游戏中的网关线程池是如何创建的](https://github.com/AwakenCN/Almost-Famous/blob/2b8e16c9476364d7a95aef228655382ac6a199e0/famous-common/src/main/java/com/noseparte/common/thread/ThreadPool.java#L70)

> **相关博文：友情链接**
[一次Java线程池误用引发的血案和总结](https://zhuanlan.zhihu.com/p/32867181)
[Java中线程池，你真的会用吗？](https://blog.csdn.net/hollis_chuang/article/details/83743723)