#### IntentService

简介：继承自Service,可以做耗时任务的Service。

1. ##### 使用

   - 创建Service继承IntentService

     创建 MyIntentService 继承IntentService ，实现 onHandleIntent方法。

     ```
     public class MyIntentService extends IntentService {
     
         private String TAG = getClass().getSimpleName();
     
     
         public MyIntentService() {
             super("MyIntentService");
         }
     
         @Override
         protected void onHandleIntent(Intent intent) {
             Log.d(TAG, "onHandleIntent BEGIN");
             try {
                 Thread.sleep(11000);
             } catch (InterruptedException e) {
                 e.printStackTrace();
             }
            String value= intent.getStringExtra("key");
             Log.d(TAG, "onHandleIntent " +value);
             Log.d(TAG, "onHandleIntent END");
         }
     
         @Override
         public void onCreate() {
             super.onCreate();
             Log.d(TAG, "onCreate");
         }
     
         @Override
         public void onStart(Intent intent, int startId) {
             super.onStart(intent, startId);
             Log.d(TAG, "onStart");
         }
     
         @Override
         public int onStartCommand(Intent intent, int flags, int startId) {
             Log.d(TAG, "onStartCommand");
             return super.onStartCommand(intent, flags, startId);
         }
     
         @Override
         public void onDestroy() {
             super.onDestroy();
             Log.d(TAG, "onDestroy");
         }
     
     ```

   - 开启Service

     ```
     public class MainActivity extends AppCompatActivity {
     
         @Override
         protected void onCreate(Bundle savedInstanceState) {
             super.onCreate(savedInstanceState);
             setContentView(R.layout.activity_main);
             Intent intent=new Intent(this,MyIntentService.class);
             intent.putExtra("key","hello");
             startService(intent);
         }
     }
     ```

2. ###### 源码解析

​         首先看一个问题，在 onHandleIntent中我先 sleep 了11秒，然后处理intent，为什么没有Anr呢。

​        第二问题，为什么会出现IntentService呢，IntentService与Service的区别。接下来通过源码去看上面两个问	题。当开启Service时会先调用 onCreate 方法，接下来就先查看onCreate的源码 看看究竟做了什么

```
 @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```

​	在onCreate中当创建了HandlerThread，HandlerThread继承自Thread，然后开启了线程，接下来调用了HandlerThread的getLooper方法，获取Looper，然后创建ServiceHandler，ServiceHandler是一个Handler类。

到这里，看到Handler，能想到IntentService内部封装了Handler的消息机制。接下来分析上述代码。

开启线程会调用HandlerThread的run方法。

```
  @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

在方法中，首先调用Looper.prepare()方法，该方法创建Looper类和消息队列，然后获取锁，获取Looper对象，并调用notifyAll(),唤醒在等待池中的等待的线程。接着调用Looper.loop()方法，开启消息队列循环处理消息。

接下来分析thread.getLooper()这句代码。

```
   public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
```

在该方法内，获取锁，并添加条件Looper是为null的话，调用wait，进入等待。刚在上述的线程的run方法中，获取到looper对象时，调用了notifyAll方法。总结上述代码。就是创建了Looper对象和消息队列，创建Handler，接下来要找到是如何调用sendMessage方法来添加消息。

接下来分析onStartCommand方法。

```
 @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
```

在该方法内调用了onStart方法。

```
@Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```

在该方法内终于找到了添加消息的方法。在这个方法内发送消息，然后然后在消息队列中一直在循环消息，若有消息便会处理，并最终调用Handler的handlerMessage方法。

```
   public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

刚方法内调用了onHandleIntent这个抽象方法。并最终由我们自己的书写的Service实现。

然后回头看我们开头的第一个问题，问什么没有anr呢，因为我们的looper.loop()方法是在HandlerThread中的run方法调用的，looper.loop()最终会调用handler的handlerMessage方法，所以handlerMessage是在HandlerThread这个工作线程中调用的，不是在主线程中调用。所以不会出现anr。

IntentService与service的区别：Service是运行在主线程中的，所以Service的方法不能做耗时任务，不然会出现anr，IntentService是专门处理在Service中做耗时任务的类。因此，若以后需要早Service中处理耗时任务不必在Service中再开线程处理，可以直接用IntentService处理。