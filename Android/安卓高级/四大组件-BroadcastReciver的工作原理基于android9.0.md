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

     该方法内最红调用ActivityManagerService的registerReceiver方法。并将ThreadActivity内部的ApplicationThread传入。

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

