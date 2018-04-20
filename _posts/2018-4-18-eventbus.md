---
layout: post
title: EventBus 3.0源码解析
comments: true
description: EventBus已经成为了我们在项目中使用率非常高的第三方开源库，每次都是简单的使用却不知道他内部到底是什么样子的，那么久跟着我的文章一步一步的剖析EventBus吧！ 
categories: code
author: jevis
---

### 综述：
 &nbsp;&nbsp;EvenBus已迥然成为了我们项目中常用的第三方开源框架，其使用简单功能强大，而且他的内部原理也十分简单，那么就让我们一起来看看他的内部实现吧！
### EventBus流程：
 ![流程图](https://github.com/greenrobot/EventBus/raw/master/EventBus-Publish-Subscribe.png)
&nbsp;&nbsp; 从图片我们可以看出来EventBus的使用真的是十分简单，事件的发布者只需要将事件"post"的到EventBus中，再由EventBus将事件传给订阅者&nbsp;&nbsp;(*图片摘抄自EventBus的[github](https://github.com/greenrobot/EventBus)*)

### 具体分析:
&nbsp;&nbsp;使用过Eventbus的人都不会对下面的那句代码陌生
```
    EventBus.getDefault().register(this);
```
&nbsp;&nbsp;这句代码便是我们在Subscriber,也就是我们经常使用的Activity或者Fragment的OnCreate的注册Eventbus的代码，那么自然地字段代码就变得很重要，因为我们猜测他一定做了很多的初始工作：

```
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```
&nbsp;&nbsp;我们看到了在getDefault()方法里面是一个Double Check的单例模式，构造了一个唯一的EventBus对象，那么我们再去看看EventBus的构造函数里面写的是什么
```
    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);//这个参数一般为null false false 
        logSubscriberExceptions = builder.logSubscriberExceptions; //一般情况为true
        logNoSubscriberMessages = builder.logNoSubscriberMessages;//一般情况为true
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;//一般情况为true 
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;//一般情况为true  
        throwSubscriberException = builder.throwSubscriberException;//一般情况为false 
        eventInheritance = builder.eventInheritance;//一般情况为true
        executorService = builder.executorService;//一个newCachedThreadPool()  
    }

```

&nbsp;&nbsp;果不其然，这里做了很多的初始化的事情，用的是EventBusBuilder，一个创建EventBus对象的Builder， 我参照其中的参数值在上面的注释中注释上了一般情况下的默认值。这里还出现了一个SubscriberMethodFinder类,我们来看一下面的三个属性，我们后面要讲到
```
 private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
 private List<SubscriberInfoIndex> subscriberInfoIndexes;
 private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
```



&nbsp;&nbsp;初始化都弄完啦，那么我们在先来看register方法

```
   public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();//获取到传过来的类对象
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);//获取类中的订阅方法
        synchronized (this) {//循环所有的订阅方法完成订阅
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
&nbsp;&nbsp;我们看到了register方法里面做了两件重要的事情，第一便是获取类中的订阅方法，第二便是将所有的方法完成订阅。那么我们先来看第一个获取订阅方法，那么他是怎么获取的呢又是什么样子的方法才是订阅方法呢。我们继续往下看subscriberMethodFinder.findSubscriberMethods这个方法里写了什么
```
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);//先在方法缓存区里面寻找有没有订阅方法
        if (subscriberMethods != null) {
            return subscriberMethods;//如果有就返回这些方法
        }
     //根据ignoreGeneratedIndex来判断使用什么方法来获取订阅方法
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        //如果没有找到 就抛异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
        //有的话就加入方法缓存区并返回
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
&nbsp;&nbsp;在上面的代码中我们见到了METHOD_CACHE这个没有第一个没有将的Map，他的键是一个类对象，值是一个List，里面存储的是订阅方法。那么就把这个称之为方法缓存区。还有呢就是findUsingReflection和findUsingInfo，这两个都是获取订阅方法，他们有什么差异呢，为什么ignoreGeneratedIndex的默认值是false呢？那我们带着疑惑来看这个两个方法的代码吧。
```
  private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```
&nbsp;&nbsp;通过方法的名字大家也猜测到了这个方法里面使用的是反射来获取subscriberClass中的订阅方法的。那么这里面还有一个prepareFindState和initForSubscriber方法。我们继续来看他们的代码。
```
   private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {//初始化设置成null
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```
&nbsp;&nbsp;这个方法就是得到了一个来装订阅方法的一个FindState类，初始设置成null。
```
    void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```
&nbsp;&nbsp;initForSubscriber方法将FindState中的subscriberClass赋值，并将subscriberInfo设置为null

    this.subscriberClass = clazz = subscriberClass

&nbsp;&nbsp;那么剩下的便是反射获取订阅方法最主要的方法findUsingReflectionInSingleClass啦
```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();//得到类中所有的非继承的方法
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();////得到类中所有的方法
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {//进行方法的循环
            int modifiers = method.getModifiers();//得到方法的修饰 类似public private等
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            //如果方法不是public的并且不是abstract和static的
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {//如果方法只有一个参数
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                    //如果方法的注释是@Subscribe
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                           //获取这个方法的订阅线程 通过(threadMode = ThreadMode.MAIN)来设置
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));//将方法，参数类型和运行线程，优先级和是不是粘性事件保存起来
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {//否则报参数不唯一的错误
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            //否则报类不是public 或者类是静态抽象的错误
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```
&nbsp;&nbsp;通过上面的代码注释我们就又明白了什么叫做订阅方法(虽然你们早就知道啦)，便是下面代码中呈现的方法
```
 @Subscribe(threadMode = ThreadMode.MAIN,priority = 1,sticky = true)
 public void onMessageEvent(MessageEvent event) {/* Do something */};
```
&nbsp;&nbsp;我们接下来看看findUsingInfo方法,其实这个方法是通过apt(Annotation Processing Tool )方法来得到所有的订阅方法的。
```
  private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);//看这里 通过apt获取订阅方法
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {//否则通过反射得到订阅方法
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

&nbsp;&nbsp;我们来进入getSubscriberInfo方法里面看看他的逻辑
```
 private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {//先判断缓存中是否存在
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {//subscriberInfoIndexes 是通过apt处理器自动生成的文件
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```
&nbsp;&nbsp;通过这一系列，我们便得到了订阅类中的订阅方法，但是这还远远不过，因为我们还没有subscribe的，这也就是我们一开始留下的坑，现在来填上吧！
```
    // Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;//获取订阅方法中的参数类型
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);//构建一个订阅事件类
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);//先根据这个参数类型来获取Subscription
        if (subscriptions == null) {//如果是空的话说明没有注册过这个参数类型的Subscription
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {//注册过 抛异常 已经被注册过
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        //下面是根据所有订阅方法的优先级进行排序
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
         //获取typesBySubscriber中的订阅事件
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {//如果事件是null的话 就添加到typesBySubscriber中
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
    
        if (subscriberMethod.sticky) {//如果是粘性事件
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }

```
&nbsp;&nbsp;最后让我们来看事件发送的post方法,这里面主要是根据线程来见里面所有的event全都发送给订阅者。
```
 public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();//获取当前线程
        List<Object> eventQueue = postingState.eventQueue;//获取当前线程的eventQueue
        eventQueue.add(event);//将事件发送到queue中

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {//如果eventQueue不为empty 就将其中的event一个一个的发送出去
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }

```
&nbsp;&nbsp;以上呢就是我对eventbus的源码分析，还有好多地方没有分析到，只分析了订阅事件是如何创建，添加到eventQueue中，又如何将eventQueue中的订阅消息发送给订阅者。就是抛砖引玉,让大家有了一个笼统的概念可以继续看eventbus的代码，如有分析的不恰当的地方还请大家指出来。