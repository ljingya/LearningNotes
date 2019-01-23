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

##### 三、源码分析

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

**typesBySubscriber:**Map<Object, List<Class<?>>>对象。该Map以Activity实例为key，以事件类型的Class对象为集合的value，组成映射。

**stickyEvents：**Map<Class<?>, Object>对象。存储粘性事件的Map。以事件类型的Class对象为key，以事件类型为value组成映射。



调用register方法。

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

在该方法内主要做以下事情：

###### 通过SubscriberMethodFinder#findSubscriberMethods方法查找对象内，添加Subscribe注解的方法。然后调用subscribe方法进行订阅。下面对该方法内的逻辑进行分析。

SubscriberMethodFinder#findSubscriberMethods代码如下：

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

SubscriberMethodFinder#findSubscriberMethods方法中先从METHOD_CACHE这个map对象缓存中获取SubscriberMethod的集合。

ignoreGeneratedIndex初始化时false,因此调用SubscriberMethodFinder#findUsingInfo方法获取到SubscriberMethod的集合，并添加到METHOD_CACHE缓存中。

SubscriberMethodFinder#findUsingInfo方法

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

在该方法中先调用SubscriberMethodFinder#prepareFindState方法获取FindState对象。FindState对象缓存在FIND_STATE_POOL这个数组中，每次调用时，会将这个对象数组清空，重新创建一个FindState对象。

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

接着调用FindState#initForSubscriber初始化数据，SubscriberInfo对象初始化为null。

```
  void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```

然后调用SubscriberMethodFinder#getSubscriberInfo方法获取FindState内部的SubscriberInfo对象。

由于FindState内部的SubscriberInfo对象初始化为null,并且subscriberInfoIndexes在EventBus对象创建的时候是为null,因此FindState内部的SubscriberInfo对象返回null。

```
 private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
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

由于SubscriberMethodFinder#getSubscriberInfo返回为null.因此调用SubscriberMethodFinder#findUsingReflectionInSingleClass方法，在该方法内通过反射获取注册的实例的所有方法，并获取方法上的注解值，并最终将方法，运行线程的类型等组装到SubscriberMethod对象中，并保存在FindState中。

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

最后调用SubscriberMethodFinder#getMethodsAndRelease方法。将保存在FindState对象中的SubscriberMethod拷贝一份，然后清空FindState对象。

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

###### 调用subscribe方法订阅

将注册的实例和 SubscriberMethod 对象构建出一个 Subscription 对象，用于保存注册的实例，以及注解方法，参数等。然后将注解方法的参数的Class对象与Subscription的集合形成映射，保存在 subscriptionsByEventType这个map中。然后按照事件设置的优先级大小对保存在Subscription集合中的信息排序。然后将注册实例与注册方法参数对象集合保存在 typesBySubscriber这个map 中。最后判断该事件是否是粘性事件。

粘性事件与非粘性事件：指粘性事件可以先发送，后注册，也会受到发送的信息，非粘性事件是必须先注册事件，后发送。如果是粘性事件调用checkPostStickyEventToSubscription。最后直接调用postToSubscription这个发送的方法。

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



   - ###### 发布分析

     发布事件调用的是以下代码,获取EventBus的单例并调用post方法。

     ```
      EventBus.getDefault().post("Hello");
     ```

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

     post方法内通过currentPostingThreadState这个ThreadLocal对象获取存储在内部的PostingThreadState对象。获取事件队列，并将发送的事件信息存储在队列中。

     ```
     private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
             @Override
             protected PostingThreadState initialValue() {
                 return new PostingThreadState();
             }
         };
     ```

     判断事件是否在发送，然后判断是否是主线程设置到PostingThreadState中并循环调用postSingleEvent。

     在该方法中首先获取发布事件参数的Class对象，接着调用EventBus#postSingleEventForEventType方法。最终调用EventBus#postToSubscription方法。

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

     ThreadMode：POSTING，默认值，表示发送事件和接受事件在统一线程。在此状态下调用EventBus#invokeSubscriber方法，在invokeSubscriber方法中通过Subscription中保存的方法对象，通过反射调用添加了Subcribe注解的方法。

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

     最终调用HandlerPoster#enqueue方法，然后调用sendMessage方法，最终调用handleMessage方法，最终调用EventBus#invokeSubscriber方法，在invokeSubscriber方法中通过Subscription中保存的方法对象，通过反射调用添加了Subcribe注解的方法。。

     其他几种基本类似，就不一一说了。

   - ###### 总结

     ###### 1. EventBus通过反射从注册的实例中获取到注解方法的方法名，注解参数的Class对象，方法参数等值，并保存起来，当调用post方法时，通过参数的Class对象获取保存起来的对应的信息。并最终通过ThreadMode的类型分别调用不同的发布事件的方式。