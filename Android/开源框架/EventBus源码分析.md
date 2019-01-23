##### EventBus源码分析

##### 一、简述

EventBus 是安卓中的一个事件总线库，可用于替代广播，Handler和用于组件化中组件间通信的库。

![](https://raw.githubusercontent.com/ljingya/LearningNotes/master/Image/EventBus.jpg)

这是EventBus的Github上的一张介绍图，从图中可以理解EventBus的工作流程，发布者即 Publisher 发布事件到EventBus中，通过EventBus将事件传递给观察者即Suncriber。

##### 二、EventBus的使用

1. ###### 添加依赖

​    在 module的gradle中添加

```
implementation 'org.greenrobot:eventbus:3.1.1'
```

2. ###### 定义事件类型

定义一个实体类，作为传递的事件。

```
public class EventMessageType {

    private int type;

    private String data;

    public int getType() {
        return type;
    }

    public void setType(int type) {
        this.type = type;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
}
```

###### 3. 事件注册

在Actiivty的onCreate中调用 **EventBus.getDefault().register(this)**，将该Activity的实例注册到EventBus中，然后需要为接收事件的方法添加 **@Subscribe**注解。在该注解类型中可以设置接收事件所在的线程，优先级，以及是否是粘性事件。当页面销毁的时候需要调用 **EventBus.getDefault().unregister(this)**  解绑注册。

```
public class EventBusAct extends AppCompatActivity implements OnClickListener {

    private TextView tvDesc;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.act_event);
        tvDesc = findViewById(R.id.tv_desc);
        tvDesc.setOnClickListener(this);
        EventBus.getDefault().register(this);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(String msg) {
        tvDesc.setText(msg);
    }

    @Override
    protected void onDestroy() {
        EventBus.getDefault().unregister(this);
        super.onDestroy();
    }

    @Override
    public void onClick(View v) {
        Intent intent = new Intent();
        switch (v.getId()) {
            case R.id.tv_desc:
                intent.setClass(this, EventBusSecAct.class);
                break;
        }
        startActivity(intent);
    }
}
```

4. ###### 发布事件

发布事件通过EventBus发布一个事件，事件类型可以是基本类型，String类型，或者自定义的JavaBean。

```
 EventBus.getDefault().post("Hello");
```

使用EventBus就是以上步骤，先注册事件，然后发布事件。在注册事件的地方便能收到发布事件时携带的数据。接下来我们就进入源码分析。理解EventBus内部是如何工作。

##### 三、源码分析-注册流程

###### 3.1 EventBus#getDefault方法

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

EventBus首先通过getDefault方法获取到单例的EventBus,该方法内做了双重判断，由于加了同步锁会对性能产生影响，这样做可以优化性能。接下来看EventBus的构造方法。

```
 public EventBus() {
        this(DEFAULT_BUILDER);
    }
```

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
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```

在构造方法内，通过EventBusBuilder类初始化了如下数据:

**subscriptionsByEventType:** Map<Class<?>, CopyOnWriteArrayList<Subscription>>对象。该Map以事件类型的Class对象为key，以 CopyOnWriteArrayList<Subscription>集合为value。Subscription保存了Activity实例，以及注解的方法名称，以及事件类型的Class对象，及注解的值，使用CopyOnWriteArrayList保证了线程安全。

**typesBySubscriber:** Map<Object, List<Class<?>>>对象。该Map以Activity实例为key，以事件类型的Class对象为集合的value，组成映射。

**stickyEvents：**Map<Class<?>, Object>对象。存储粘性事件的Map。以事件类型的Class对象为key，以事件类型为value组成映射。

###### 3.2 EventBus#register方法

调用register方法，在该方法内获取Activity实例的Class对象，然后调用SubscriberMethodFinder#findSubscriberMethods方法获取SubscriberMethod方法的集合。然后遍历调用EventBus#subscribe方法。

```
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

###### 3.3 SubscriberMethodFinder#findSubscriberMethods方法

先从METHOD_CACHE这个map对象缓存中获取泛型为SubscriberMethod的集合。不为空直接返回该对象。不为空走以下逻辑。

由于ignoreGeneratedIndex初始化时false,因此调用SubscriberMethodFinder#findUsingInfo方法获取到SubscriberMethod的集合，并添加到METHOD_CACHE缓存中。

```
 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

###### METHOD_CACHE

METHOD_CACHE是以Class对象为key即存储的是Activity实例的Class对象。以泛型是SubscriberMethod的集合为value,那么SubscriberMethod存储的是什么值呢。

```
Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
```

###### SubscriberMethod

SubscriberMethod类中存储了接受事件的方法名称的反射类型，线程模式，事件类型的Class对象，优先级，是否是粘性事件。

```
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;
    ....
    }
```

###### 3.4 SubscriberMethodFinder#findUsingInfo方法

先调用SubscriberMethodFinder#prepareFindState方法获取FindState对象。然后将Activity的实例的Class对象缓存在FindState对象中。由于getSubscriberInfo方法返回为null,所以最终调用findUsingReflectionInSingleClass方法并将FindState对象传入。

```
 private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

###### 3.5 FindState#prepareFindState方法

在 prepareFindState 方法中，FIND_STATE_POOL是FindState类型的，大小为POOL_SIZE=4的长度数组，每次调用时，会找到这个数组内为空的对象，并将数组内该索引清空，返回当前索引的对象，如果这个数组内为空重新创建一个FindState对象。

```
private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```

###### 3.6 FindState#initForSubscriber

将Actiivty实例的Class对象缓存在FindState内。

```
  void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```

###### 3.7 SubscriberMethodFinder#findUsingReflectionInSingleClass

在该方法内通过反射获取Activity的实例的Class对象的所有方法，并获取方法上的注解值，并最终将方法，运行线程的类型等组装到SubscriberMethod对象中，并保存在FindState中。

```
    private void findUsingReflectionInSingleClass(FindState findState) {
               Method[] methods;
               try {
                   // This is faster than getMethods, especially when subscribers are fat classes like Activities
                   methods = findState.clazz.getDeclaredMethods();
               } catch (Throwable th) {
                   // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
                   methods = findState.clazz.getMethods();
                   findState.skipSuperClasses = true;
               }
               for (Method method : methods) {
                   int modifiers = method.getModifiers();
                   if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                       Class<?>[] parameterTypes = method.getParameterTypes();
                       if (parameterTypes.length == 1) {
                           Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                           if (subscribeAnnotation != null) {
                               Class<?> eventType = parameterTypes[0];
                               if (findState.checkAdd(method, eventType)) {
                                   ThreadMode threadMode = subscribeAnnotation.threadMode();
                                   findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                           subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                               }
                           }
                       } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                           String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                           throw new EventBusException("@Subscribe method " + methodName +
                                   "must have exactly 1 parameter but has " + parameterTypes.length);
                       }
                   } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                       String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                       throw new EventBusException(methodName +
                               " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
                   }
               }
           }
```

###### 3.8 SubscriberMethodFinder#getMethodsAndRelease方法。

将保存在FindState对象中的SubscriberMethod拷贝一份，然后清空FindState内的缓存数据，。

```
  private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
               List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
               findState.recycle();
               synchronized (FIND_STATE_POOL) {
                   for (int i = 0; i < POOL_SIZE; i++) {
                       if (FIND_STATE_POOL[i] == null) {
                           FIND_STATE_POOL[i] = findState;
                           break;
                       }
                   }
               }
               return subscriberMethods;
           }
```

###### 3.9 EventBbus#subscribe方法订阅

将Activity的实例对象和 SubscriberMethod 对象保存在 Subscription 对象中。然后将注解方法的参数的Class对象与Subscription的集合形成映射，保存在 subscriptionsByEventType这个map中。然后按照事件设置的优先级大小对保存在Subscription集合中的信息排序。然后将Activity的实例对象与事件类型的Class对象的集合保存在 typesBySubscriber这个map 中。最后判断该事件是否是粘性事件。

粘性事件与非粘性事件：指粘性事件可以先发送，后注册，也会受到发送的信息，非粘性事件是必须先注册事件，后发送。如果是粘性事件直接调用checkPostStickyEventToSubscription。然后调用postToSubscription这个发送的方法，关于postToSubscription方法稍后再发布分析中去解析。

```
 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
            Class<?> eventType = subscriberMethod.eventType;
            Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
            CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
            if (subscriptions == null) {
                subscriptions = new CopyOnWriteArrayList<>();
                subscriptionsByEventType.put(eventType, subscriptions);
            } else {
                if (subscriptions.contains(newSubscription)) {
                    throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                            + eventType);
                }
            }
    
            int size = subscriptions.size();
            for (int i = 0; i <= size; i++) {
                if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                    subscriptions.add(i, newSubscription);
                    break;
                }
            }
    
            List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
            if (subscribedEvents == null) {
                subscribedEvents = new ArrayList<>();
                typesBySubscriber.put(subscriber, subscribedEvents);
            }
            subscribedEvents.add(eventType);
    
            if (subscriberMethod.sticky) {
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

```
   private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```



###### 四、源码分析-发布流程

###### 4.1 EventBus#post

发布事件调用的是以下代码,获取EventBus的单例并调用post方法。

```
 EventBus.getDefault().post("Hello");
```

通过currentPostingThreadState这个ThreadLocal对象获取存储在内部的PostingThreadState对象。通过PostingThreadState获取其内部的eventQueue集合。

判断事件是否在发送，然后判断是否是主线程，设置到PostingThreadState对象中，循环调用EventBus#postSingleEvent方法，直到eventQueue集合为空。

```
/** Posts the given event to the event bus. */
    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

```
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };
```

###### 4.2 PostingThreadState

PostingThreadState是EventBus内部的一个类，存储Subscription类以及事件类型对象及其他信息。上文提到Subscription中保存了注册的Actvity的实例对象和存储了事件类型的Class对象，注解值，及方法名的SubscriberMethod对象。

```
   final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
```

###### 4.3 EventBus#postSingleEvent

在该方法中首先获取事件类型的Class对象，接着调用EventBus#postSingleEventForEventType方法。将事件类型对象，以及PostingThreadState和事件类型的Class对象传进去。

```
 private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

###### 4.4 EventBus#postSingleEventForEventType

在该方法内通过subscriptionsByEventType这个map获取到Subscription对象的集合，subscriptionsByEventType这个map在上文的subcribe方法中提到存储了事件类型的Class对象以及Subscription对象的集合。然后将事件类型对象和Subscription对象设置给PostingThreadState对象。并调用postToSubscription。

```
 private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```

###### 4.5 EventBus#postToSubscription

```
  private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

ThreadMode：POSTING，默认值，表示发送事件和接受事件在统一线程。在此状态下调用EventBus#invokeSubscriber方法，在invokeSubscriber方法中通过反射调用保存Subscription对象中Activity实例中的注解方法。

```
void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```

ThreadMode：MAIN，首先判断是否是主线程，若是主线程直接调用，此种情况与ThreadMode为POSTING一致。若不是则调用mainThreadPoster这个Poster#enqueue方法，具体实现是在EventBus初始化时创建Poster的实现类。

```
 mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
```

在MainThreadSupport中的内部类AndroidHandlerMainThreadSupport中通过主线程的Looper对象创建HandlerPoster即是Handler类。

```
@Override
        public Poster createPoster(EventBus eventBus) {
            return new HandlerPoster(eventBus, looper, 10);
        }
```

最终调用HandlerPoster#enqueue方法，然后调用sendMessage方法，最终调用handleMessage方法，最终调用EventBus#invokeSubscriber方法，在invokeSubscriber方法中通过反射调用Subscription中保存的Activity实例的添加注解的方法。

其他几种基本类似，就不一一说了,最终调用与其他两种一致。

   - ###### 总结

     ###### EventBus通过反射处理注解，从注册的实例的Class对象中获取到注解方法的方法名，事件类型的Class对象及注册的Activity实例等值，并保存起来，当调用post方法时，通过事件类型的Class对象获取保存起来的对应的信息。并最终通过反射调用保存的Activity实例的注解方法。