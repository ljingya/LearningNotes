#### 四大组件-Service的工作原理基于android9.0

安卓中启动Service的方式有两种，startService及bindService，因此这篇文章会基于这两种方式分析安卓9.0的源码中对于这两种方式的实现。另外只分析具体的流程，不对具体的细节做分析，这也是阅读源码需要避免的错误方式。

1. ##### startService方式

   调用startService时，会调用ContextWrapper方法的startService

   ```
   	@Override
       public ComponentName startService(Intent service) {
           return mBase.startService(service);
       }
   ```

   mBase是Context类，另外它的具体实现是ContextImpl类，ContextWrapper是一个包装类，像我们的Service，Activity都继承该类，ContextWrapper会将与Context的相关调用转发给ContextImpl类。

   - ContextImpl的startService

     在该方法内调用startServiceCommon，第二个参数传的false，使用这种方式启动的会默认将service设置为后台服务。在startServiceCommon内部首先通过ActivityManager.getService()获取IActivityManager，IActivityManager是一个Binder类型的对象，它的具体实现是ActivityManagerService。

     ```
       @Override
         public ComponentName startService(Intent service) {
             warnIfCallingFromSystemProcess();
             return startServiceCommon(service, false, mUser);
         }
     ```

     ```
      private ComponentName startServiceCommon(Intent service, boolean requireForeground,
                 UserHandle user) {
             try {
             
                 validateServiceIntent(service);
                 service.prepareToLeaveProcess(this);
                 ComponentName cn = ActivityManager.getService().startService(
                     mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                                 getContentResolver()), requireForeground,
                                 getOpPackageName(), user.getIdentifier());
             ......
                 return cn;
             } catch (RemoteException e) {
                 throw e.rethrowFromSystemServer();
             }
         }
     ```

   - ###### ActivityManagerService的startService

     该方法内继续调用了mServices即ActiveServices的startServiceLocked方法。

     ```
      @Override
         public ComponentName startService(IApplicationThread caller, Intent service,
                 String resolvedType, boolean requireForeground, String callingPackage, int userId)
                 throws TransactionTooLargeException {
             enforceNotIsolatedCaller("startService");
            ....
             synchronized(this) {
                 final int callingPid = Binder.getCallingPid();
                 final int callingUid = Binder.getCallingUid();
                 final long origId = Binder.clearCallingIdentity();
                 ComponentName res;
                 try {
                     res = mServices.startServiceLocked(caller, service,
                             resolvedType, callingPid, callingUid,
                             requireForeground, callingPackage, userId);
                 } finally {
                     Binder.restoreCallingIdentity(origId);
                 }
                 return res;
             }
         }
     ```

   - ###### ActiveServices的startServiceLocke

     在该方法内调用startServiceInnerLocked方法,接着调用bringUpServiceLocked调起服务的方法,在bringUpServiceLocked方法中分两种情况，一种是Service已经启动，Service未启动，Service已经启动会调用sendServiceArgsLocked方法，Service未启动会调用realStartServiceLocked

     ```
     ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
                 int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
                 throws TransactionTooLargeException {
                 ....
                 ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
             return cmp; 
                 }
     ```

     ```
     ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
                 boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
                 ...
                    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
                 ...
                 }
     ```

     ```
     private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
                 boolean whileRestarting, boolean permissionsReviewRequired)
                 throws TransactionTooLargeException {
                  if (r.app != null && r.app.thread != null) {
                 sendServiceArgsLocked(r, execInFg, false);
                 return null;
             }
             ...
             
             if (!isolated) {
                 app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
                 if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                             + " app=" + app);
                 if (app != null && app.thread != null) {
                     try {
                         app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                         realStartServiceLocked(r, app, execInFg);
                         return null;
                     } catch (TransactionTooLargeException e) {
                         throw e;
                     } catch (RemoteException e) {
                         Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                     }
     
                     // If a dead object exception was thrown -- fall through to
                     // restart the application.
                 }
             } 
             ....
                 }
     ```

     ######      **Service**未启动

     ​	调用app.thread的scheduleCreateService方法，app.thread获取的IApplicationThread的Binder类型的对象，它的具体实现是在ActivityThread的ApplicationThread中

     ```
     private final void realStartServiceLocked(ServiceRecord r,
                 ProcessRecord app, boolean execInFg) throws RemoteException {
                 ...
                 app.thread.scheduleCreateService(r, r.serviceInfo,
                         mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                         app.repProcState);
                 ...
                 }
     ```

     ​	ApplicationThread的scheduleCreateService方法中，调用sendMessage，sendMessage中调用内部的Handler的sendMessage方法，最终回调到内部mH这个Handler的handleMessage方法，

     ```
      public final void scheduleCreateService(IBinder token,
                     ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
                 updateProcessState(processState, false);
                 CreateServiceData s = new CreateServiceData();
                 s.token = token;
                 s.info = info;
                 s.compatInfo = compatInfo;
     
                 sendMessage(H.CREATE_SERVICE, s);
             }
     ```

     ```
     private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
         if (DEBUG_MESSAGES) Slog.v(
             TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
             + ": " + arg1 + " / " + obj);
         Message msg = Message.obtain();
         msg.what = what;
         msg.obj = obj;
         msg.arg1 = arg1;
         msg.arg2 = arg2;
         if (async) {
             msg.setAsynchronous(true);
         }
         mH.sendMessage(msg);
     }
     ```

     在handleMessage中有这段处理H.CREATE_SERVICE的代码，调用了handleCreateService方法，在该方法中获取ClassLoader类，并实例化Service对象，创建上下文Context，绑定Application。接着调用service.onCreate()。到这里Service的启动就分析完毕了。

     ```
     case CREATE_SERVICE:
                         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                         handleCreateService((CreateServiceData)msg.obj);
                         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                         break;
     ```

     ```
     private void handleCreateService(CreateServiceData data) {
             // If we are getting ready to gc after going to the background, well
             // we are back active so skip it.
             unscheduleGcIdler();
     
             LoadedApk packageInfo = getPackageInfoNoCheck(
                     data.info.applicationInfo, data.compatInfo);
             Service service = null;
             try {
                 java.lang.ClassLoader cl = packageInfo.getClassLoader();
                 service = packageInfo.getAppFactory()
                         .instantiateService(cl, data.info.name, data.intent);
             } catch (Exception e) {
                 if (!mInstrumentation.onException(service, e)) {
                     throw new RuntimeException(
                         "Unable to instantiate service " + data.info.name
                         + ": " + e.toString(), e);
                 }
             }
     
             try {
                 if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
     
                 ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
                 context.setOuterContext(service);
     
                 Application app = packageInfo.makeApplication(false, mInstrumentation);
                 service.attach(context, this, data.info.name, data.token, app,
                         ActivityManager.getService());
                 service.onCreate();
                 mServices.put(data.token, service);
                 try {
                     ActivityManager.getService().serviceDoneExecuting(
                             data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                 } catch (RemoteException e) {
                     throw e.rethrowFromSystemServer();
                 }
             } catch (Exception e) {
                 if (!mInstrumentation.onException(service, e)) {
                     throw new RuntimeException(
                         "Unable to create service " + data.info.name
                         + ": " + e.toString(), e);
                 }
             }
         }
     ```

     ​     **Service已启动**

     上面提到Service已启动的情况下会调用sendServiceArgsLocked(r, execInFg, false)方法，  r.app.thread是ProcessRecord类中的IApplicationThread的变量，即调用ActivityThread中的ApplicationThread中的scheduleServiceArgs方法。

     ```
     private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
                 boolean oomAdjusted) throws TransactionTooLargeException {
                 ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
             slice.setInlineCountLimit(4);
             Exception caughtException = null;
             try {
                 r.app.thread.scheduleServiceArgs(r, slice);
             } 
                 }
     ```

     ApplicationThread的scheduleServiceArgs

     在该方法中最终也会调用到Activity的内部的mH这个Handler类的handleMessage方法。

     ```
       public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
                 List<ServiceStartArgs> list = args.getList();
     
                 for (int i = 0; i < list.size(); i++) {
                     ServiceStartArgs ssa = list.get(i);
                     ServiceArgsData s = new ServiceArgsData();
                     s.token = token;
                     s.taskRemoved = ssa.taskRemoved;
                     s.startId = ssa.startId;
                     s.flags = ssa.flags;
                     s.args = ssa.args;
     
                     sendMessage(H.SERVICE_ARGS, s);
                 }
             }
     ```

       handleMessage方法:该方法中调用handleServiceArgs方法，在handleServiceArgs中，先判断Service是否被存在，若存在调用onStartCommand方法。

     ```
     case SERVICE_ARGS:
                         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceStart: " + String.valueOf(msg.obj)));
                         handleServiceArgs((ServiceArgsData)msg.obj);
                         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                         break;
     ```

     ```
      private void handleServiceArgs(ServiceArgsData data) {
             Service s = mServices.get(data.token);
             if (s != null) {
                 try {
                     if (data.args != null) {
                         data.args.setExtrasClassLoader(s.getClassLoader());
                         data.args.prepareToEnterProcess();
                     }
                     int res;
                     if (!data.taskRemoved) {
                         res = s.onStartCommand(data.args, data.flags, data.startId);
                     } else {
                         s.onTaskRemoved(data.args);
                         res = Service.START_TASK_REMOVED_COMPLETE;
                     }
     
                     QueuedWork.waitToFinish();
     
                     try {
                         ActivityManager.getService().serviceDoneExecuting(
                                 data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                     } catch (RemoteException e) {
                         throw e.rethrowFromSystemServer();
                     }
                     ensureJitEnabled();
                 } catch (Exception e) {
                     if (!mInstrumentation.onException(s, e)) {
                         throw new RuntimeException(
                                 "Unable to start service " + s
                                 + " with " + data.args + ": " + e.toString(), e);
                     }
                 }
             }
         }
     ```

2. ##### bindService

当调用bindService时回调ContextWrapper中bindService方法，该方法中最终也是调用ContextImpl中的bindService方法。

```
@Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }
```

- ContextImpl的bindService

  在该方法中调用bindServiceCommon方法

```
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, mMainThread.getHandler(), getUser());
}
```