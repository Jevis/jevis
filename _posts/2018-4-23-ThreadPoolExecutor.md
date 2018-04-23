---
layout: post
title: ThreadPoolExecutor线程池之123
comments: true
description: 我们在上面的博客已经讲了Thread和Runnable的区别，如何我们在项目中大量的使用他们的话，会导致大量的资源被占用，甚至会引发OOM，所以我们要在项目中使用线程池！ 
categories: android
author: jevis
---
# ThreadPoolExecutor


## 简述：
&nbsp;&nbsp;有的时候，系统处理很多任务，如何这些任务要是都是通过new Thread来做的话，系统就不得不常常的创建之后还要销毁Thread，这个是非常消耗时间而且还占用资源，所以我们通过创建线程池来管理我们的线程。

## ThreadPoolExecutor:

&nbsp;&nbsp;我们是首先来看一下他的构造函数,并且讲解一下里面参数的意思
```
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
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

 - corePoolSize
&nbsp;&nbsp;核心线程数。当我们向线程池中提交新的任务的时候，线程池会创建一个新的线程来执行这个任务，直到线程数等于corePoolSize，默认情况下核心线程会一直存活下去的。
 - maximumPoolSize
&nbsp;&nbsp;最大线程数。要是阻塞队列满了的话，继续添加新的线程的话，并且当前线程小于maximumPoolSize，那么就会创建新的线程来执行这个任务
 - keepAliveTime
&nbsp;&nbsp;线程允许空闲状态存活的时间，前提是此线程不是核心线程。
 - unit
&nbsp;&nbsp;这个参数是keepAliveTime的时间的单位，例如MICROSECONDS，SECONDS，MINUTES等等
 - workQueue
&nbsp;&nbsp;阻塞队列。用来存储等待执行的任务的阻塞队列，系统给我们提供了一下几个阻塞队列：ArrayBlockingQueue,LinkedBlockingQuene,SynchronousQuene和priorityBlockingQuene(我们在后面会讲到)
 - threadFactory
&nbsp;&nbsp;线程工厂。默认的是DefaultThreadFactory，里面定义了线程计数，线程命名规则。当然这些我们都可以自定义。
 - handler
&nbsp;&nbsp;饱和策略。我们可以实现RejectedExecutionHandler重写rejectedExecution方法来自定义，当然系统也给我们提供了几个：AbortPolicy，
 DiscardPolicy，DiscardOldestPolicy，CallerRunsPolicy（我们在后面也会讲到）

&nbsp;&nbsp;我们前面笼统的提到了很多概念我猜测很多人现在都不太懂，那我们一点一点的来深入吧。
## BlockingQueue
&nbsp;&nbsp;BlockingQueue接口继承了Queue接口，里面有比较重要的几个方法，分别是offer方法，poll方法，普通方法，take方法，我们看看他的区别：
|----|抛出异常 |特殊值|阻塞|超时|
|----|:-------:|:-----:|:--:|:--:|
|插入|add(e)   |offer(e)|put(e)|offer(e.time,unit)|
|移除|remove() |poll()|take()|poll(time,unit)|
|检查|element() |peek()|不可用|不可用|
 
add方法会在插入数据时，插入失败就会抛异常，offer会在插入失败时返回false，put方法会在当队列存储对象达到限定值阻塞线程，而在队列不为空的时唤醒被take方法所阻塞的线程。
 
### 常见BlockingQueue的实现

 - ArrayBlockingQueue

&nbsp;&nbsp;他是一个底层是由数组实现的有界的FIFO(先进先出)的阻塞队列。我们看一下他的构造函数
```
  public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```
&nbsp;&nbsp;参数1：指定queue的capacity(容量),并且一旦设定则无法更改.
&nbsp;&nbsp;参数2:一个默认是false的bool值，指的是创建的是不是公平锁，如果为true，则按照 FIFO 顺序访问插入或移除时受阻塞线程的队列；如果为 false，则访问顺序是不确定的
```
 public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

```

 - LinkedBlockingQueue

 &nbsp;&nbsp;一个基于链表的阻塞队列，他的内部维持着一个链表队列用于存储。他的容量也是有固定大小的，如果不设置的情况下默认值是Integer.MAX_VALUE，他之所以能够高效的处理高并发的数据。是因为他的采用了两个独立的锁来控制数据同步
 ```
     /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
 ```

 - PriorityBlockingQueue
&nbsp;&nbsp;一个基于优先级的队列。他的优先级通过构造函数传入的Compator对象来决定，当时这个队列并不会阻塞数据的生产者，当数据队列中没有可消费的数据的时候，会阻塞消费者。那么我们就需要注意生成者的生成数据的速率不能高于消费速率，不然会消耗可用的堆内存。

 - SynchronousQueue
 &nbsp;&nbsp;这个队列的神奇之处在于它内部没有容器，什么意思呢，就是当你的生产线程(put)成产啦，如果没有消费者消费(take)，那么此生产线程就会被阻塞，知道有消费者调用take的时候。反过来先take等待put也是同理


### ThreadFactory
 &nbsp;&nbsp;ThreadFactory接口里面的内容很简单，我们看一下源码
```
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```
&nbsp;&nbsp;他的实现也是非常简单，只需要我们重写里面的newThread方法返回一个Thread就可以啦，我们可以在里面设置我们自己的命名格式等等。那我们先看看JDK中的默认实现DefaultThreadFactory:
```
    /**
     * The default thread factory.
     */
    private static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);//创建生成线程的名字
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```
### handler 
&nbsp;&nbsp;既然要说这个饱和策略那就先说一个任务是怎么被添加的吧！.一个新的任务通过 execute(Runnable) 方法被添加到线程池，任务就是一个 Runnable 类型的对象，任务的执行方法就是 Runnable 类型对象的 run() 方法。那么当一个任务通过 execute(Runnable) 方法欲添加到线程池时，线程池采用的策略如下：

 **1. 当线程池中的数量小于corePoolSize的时候，即使你的线程池中的线程都处于空闲状态，也要开启一个新的线程来处理新的任务。
 - 当你线程池的数量等于corePoolSize的时候，且你的BlockQueue队列没有满的时候，那么任务被加入缓冲队列。
 - 当你的线程池的数量大于corePoolSize的时候，并且你的BlockQueue队列满的时候，但是线程池的数量小于maximumPoolSize的时候，会创建新的线程处理新的任务。
 - 当你的线程池的数量大于corePoolSize的时候，并且你的BlockQueue队列满的时候，但是线程池的数量等于maximumPoolSize的时候，我们的handler出场啦，就有这个饱和策略来处理新的任务**
 
&nbsp;&nbsp;我们有四种饱和模式
 - ThreadPoolExecutor.AbortPolicy() 
 &nbsp;&nbsp;&nbsp;&nbsp;对拒绝任务执行抛弃处理，并且抛出异常
 - ThreadPoolExecutor.CallerRunsPolicy
&nbsp;&nbsp;&nbsp;&nbsp; 这个策略重试添加当前的任务，他会自动重复调用 execute() 方法，直到成功
 - ThreadPoolExecutor.DiscardOldestPolicy()
 &nbsp;&nbsp;&nbsp;&nbsp;对拒绝任务不抛弃，而是抛弃队列里面等待最久的一个线程，然后把拒绝任务加到队列
 - ThreadPoolExecutor.DiscardPolicy
 &nbsp;&nbsp;&nbsp;&nbsp;对拒绝任务直接无声抛弃，没有异常信息。


### ThreadPoolExecutor的使用
&nbsp;&nbsp;小伙伴在看完之后肯定觉得线程池使用起来有点麻烦，难道就没有现成的让我们使用吗？当然有啦，好东西都是留在最后的吗！
 - newFixedThreadPool
&nbsp;&nbsp;固定大小的线程池，根据参数来决定生成的核心线程数量和最大线程数量，内部使用的是LinkedBlockingQueue
```
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
 - newCachedThreadPool
&nbsp;&nbsp;可缓存的线程池，因为内部使用的是SynchronousQueue队列，由于核心线程数为0，最大线程是为Integer.MAX_VALUE，可以让他灵活回收空闲线程，若无可回收，则新建线程。
```
   public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

```

 - newScheduledThreadPool 
 &nbsp;&nbsp;根据参数来生成一个定长的线程池，由于使用的是DelayedWorkQueue，这个我们在前面没有介绍，在这里简单的说一下 他是一个无界阻塞队列，只有在延迟期满时才能从中提取元素，那么就可以用来做延迟操作或者周期性的操作啦
 ```
   public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
 ```
 
 - newSingleThreadExecutor
 &nbsp;&nbsp;他的核心线程数还有最大线程数都是1，又由于用的是LinkedBlockingQueue无参数构造，那就是默认的Integer.MAX_VALUE的个数的阻塞队列，换句话说就是这里会一个一个安装指定顺序执行任务
 ```
     public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
 ```
 
 
  &nbsp;&nbsp;当然除了这些还有好多自带呢不在一一说明，希望读者能够自己带着感悟去看源码，将会有跟深层次的理解！


