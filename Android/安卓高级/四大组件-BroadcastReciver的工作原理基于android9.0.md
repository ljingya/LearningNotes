##### 四大组件-BroadcastReciver的工作原理基于android9.0

Android中的广播分为动态广播和静态广播，静态广播需要在清单文件中注册，动态广播使用代码在需要的地方注册，这里只分析动态广播的注册过程，

1. ###### 注册过程

   动态的广播注册调用ContextWrapper的registerReceiver方法。mBase是Context类，具体的实现是在ContextImpl中。

   ```
    @Override
       public Intent registerReceiver(
           BroadcastReceiver receiver, IntentFilter filter) {
           return mBase.registerReceiver(receiver, filter);
       }
   ```

   - ContextImpl的registerReceiver方法

     该方法最终调用registerReceiverInternal方法。

     ```
     @Override
         public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
             return registerReceiver(receiver, filter, null, null);
         }
     ```

   - ContextImpl的registerReceiverInternal方法

     该方法内最终调用ActivityManagerService的registerReceiver方法。并将ThreadActivity内部的ApplicationThread传入。这个过程设计到跨进程，因此使用了IIntentReceiver这个Binder接口，它的具体实现时LoadedApk.ReceiverDispatcher.InnerReceiver，创建了ReceiverDispatcher对象，并将BroadCastReciver及InnerReceiver保存在ReceiverDispatcher对象中。

     ```
     private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
                 IntentFilter filter, String broadcastPermission,
                 Handler scheduler, Context context, int flags) {
             IIntentReceiver rd = null;
             if (receiver != null) {
                 if (mPackageInfo != null && context != null) {
                     if (scheduler == null) {
                         scheduler = mMainThread.getHandler();
                     }
                     rd = mPackageInfo.getReceiverDispatcher(
                         receiver, context, scheduler,
                         mMainThread.getInstrumentation(), true);
                 } else {
                     if (scheduler == null) {
                         scheduler = mMainThread.getHandler();
                     }
                     rd = new LoadedApk.ReceiverDispatcher(
                             receiver, context, scheduler, null, true).getIIntentReceiver();
                 }
             }
             try {
                 final Intent intent = ActivityManager.getService().registerReceiver(
                         mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                         broadcastPermission, userId, flags);
                 if (intent != null) {
                     intent.setExtrasClassLoader(getClassLoader());
                     intent.prepareToEnterProcess();
                 }
                 return intent;
             } catch (RemoteException e) {
                 throw e.rethrowFromSystemServer();
             }
         }
     ```

   - ActivityManagerService的registerReceiver方法

     在该方法中将会将远程的InnerReceiver以及BroadcastFilter保存起来。

     ```
      ...
       mRegisteredReceivers.put(receiver.asBinder(), rl);
      ....
      BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                         permission, callingUid, userId, instantApp, visibleToInstantApps);
                 if (rl.containsFilter(filter)) {
                     Slog.w(TAG, "Receiver with filter " + filter
                             + " already registered for pid " + rl.pid
                             + ", callerPackage is " + callerPackage);
                 } else {
                     rl.add(bf);
                     if (!bf.debugCheck()) {
                         Slog.w(TAG, "==> For Dynamic broadcast");
                     }
                     mReceiverResolver.addFilter(bf);
                 }
       ...
     ```


2. ###### 发布过程

   - ContexImpl的sendBroadcast

   ​	广播的发送会调用Context的sendBroadcast方法，具体实现依然在ContextImpl中，

   ​	在该方法中调用了AMS的broadcastIntent方法	。

   ```
   @Override
       public void sendBroadcast(Intent intent) {
           warnIfCallingFromSystemProcess();
           String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
           try {
               intent.prepareToLeaveProcess(this);
               ActivityManager.getService().broadcastIntent(
                       mMainThread.getApplicationThread(), intent, resolvedType, null,
                       Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                       getUserId());
           } catch (RemoteException e) {
               throw e.rethrowFromSystemServer();
           }
       }
   ```



   - ActivityManagerService的broadcastIntent

     在该方法中通过intent获取BroadcastQueue一个队列，最后调用scheduleBroadcastsLocked方法。

     ```
     ...
       final BroadcastQueue queue = broadcastQueueForIntent(intent);
                 BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                         callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                         requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
                         resultCode, resultData, resultExtras, ordered, sticky, false, userId);
                 if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing parallel broadcast " + r);
                 final boolean replaced = replacePending
                         && (queue.replaceParallelBroadcastLocked(r) != null);
                 // Note: We assume resultTo is null for non-ordered broadcasts.
                 if (!replaced) {
                     queue.enqueueParallelBroadcastLocked(r);
                     queue.scheduleBroadcastsLocked();
                 }
     ...
     ```

   - BroadcastQueue的scheduleBroadcastsLocked方法

     在该方法内发送一个BROADCAST_INTENT_MSG状态的消息。

     ```
     
         public void scheduleBroadcastsLocked() {
             if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                     + mQueueName + "]: current="
                     + mBroadcastsScheduled);
     
             if (mBroadcastsScheduled) {
                 return;
             }
             mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
             mBroadcastsScheduled = true;
         }
     ```

     BROADCAST_INTENT_MSG的消息处理如下，调用processNextBroadcast方法，并接着调用deliverToRegisteredReceiverLocked方法

     ```
      private final class BroadcastHandler extends Handler {
             public BroadcastHandler(Looper looper) {
                 super(looper, null, true);
             }
     
             @Override
             public void handleMessage(Message msg) {
                 switch (msg.what) {
                     case BROADCAST_INTENT_MSG: {
                         if (DEBUG_BROADCAST) Slog.v(
                                 TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                         processNextBroadcast(true);
                     } break;
                     case BROADCAST_TIMEOUT_MSG: {
                         synchronized (mService) {
                             broadcastTimeoutLocked(true);
                         }
                     } break;
                 }
             }
         }
     ```

   - deliverToRegisteredReceiverLocked方法

     在该方法内调用performReceiveLocked方法。

     ```
     ...
     performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                             new Intent(r.intent), r.resultCode, r.resultData,
                             r.resultExtras, r.ordered, r.initialSticky, r.userId);
     ...
     ```

   - performReceiveLocked方法

     在该方法内调用app.thread.scheduleRegisteredReceiver方法，也就是ActiivtyThread内的ApplicationThread的scheduleRegisteredReceiver方法。

     ```
     void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
                 Intent intent, int resultCode, String data, Bundle extras,
                 boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
             // Send the intent to the receiver asynchronously using one-way binder calls.
             if (app != null) {
                 if (app.thread != null) {
                     // If we have an app thread, do the call through that so it is
                     // correctly ordered with other one-way calls.
                     try {
                         app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                                 data, extras, ordered, sticky, sendingUser, app.repProcState);
                     // TODO: Uncomment this when (b/28322359) is fixed and we aren't getting
                     // DeadObjectException when the process isn't actually dead.
                     //} catch (DeadObjectException ex) {
                     // Failed to call into the process.  It's dying so just let it die and move on.
                     //    throw ex;
                     } catch (RemoteException ex) {
                         // Failed to call into the process. It's either dying or wedged. Kill it gently.
                         synchronized (mService) {
                             Slog.w(TAG, "Can't deliver broadcast to " + app.processName
                                     + " (pid " + app.pid + "). Crashing it.");
                             app.scheduleCrash("can't deliver broadcast");
                         }
                         throw ex;
                     }
                 } else {
                     // Application has died. Receiver doesn't exist.
                     throw new RemoteException("app.thread must not be null");
                 }
             } else {
                 receiver.performReceive(intent, resultCode, data, extras, ordered,
                         sticky, sendingUser);
             }
         }
     ```

   - ApplicationThread的scheduleRegisteredReceiver方法

     在该方法内调用IIntentReceiver这个Binder接口的performReceive，也就调用上文说到的LoadedAPK.ReceiverDispatcher.InnerReceiver的performReceive方法

     ```
     public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                     int resultCode, String dataStr, Bundle extras, boolean ordered,
                     boolean sticky, int sendingUser, int processState) throws RemoteException {
                 updateProcessState(processState, false);
                 receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                         sticky, sendingUser);
             }
     ```

   - InnerReceiver的performReceive方法

     在该方法内调用LoadedApk.ReceiverDispatcher的performReceive方法

     ```
      @Override
                 public void performReceive(Intent intent, int resultCode, String data,
                         Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                     final LoadedApk.ReceiverDispatcher rd;
                   ....
                         rd.performReceive(intent, resultCode, data, extras,
                                 ordered, sticky, sendingUser);
                    ...
                 }
     ```

   - ReceiverDispatcher的performReceive方法

     在该方法中调用了mActivityThread.post(args.getRunnable())，mActivityThread是一个Handler，args是Args对象，通过getRunnable获取一个Runnable对象，并执行run方法。

     ```
      public void performReceive(Intent intent, int resultCode, String data,
                     Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                 final Args args = new Args(intent, resultCode, data, extras, ordered,
                         sticky, sendingUser);
                 if (intent == null) {
                     Log.wtf(TAG, "Null intent received");
                 } else {
                     if (ActivityThread.DEBUG_BROADCAST) {
                         int seq = intent.getIntExtra("seq", -1);
                         Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction()
                                 + " seq=" + seq + " to " + mReceiver);
                     }
                 }
                 if (intent == null || !mActivityThread.post(args.getRunnable())) {
                     if (mRegistered && ordered) {
                         IActivityManager mgr = ActivityManager.getService();
                         if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                 "Finishing sync broadcast to " + mReceiver);
                         args.sendFinished(mgr);
                     }
                 }
             }
     ```

     Runnable的run方法。

     在该方法内最终调用BroadcastReciver的onReciver方法。

     ```
     ...
     try {
                             ClassLoader cl = mReceiver.getClass().getClassLoader();
                             intent.setExtrasClassLoader(cl);
                             intent.prepareToEnterProcess();
                             setExtrasClassLoader(cl);
                             receiver.setPendingResult(this);
                             receiver.onReceive(mContext, intent);
                         } catch (Exception e) {
     ...				                    
     ```




   ​		





   ​	


