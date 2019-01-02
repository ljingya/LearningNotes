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

- bindServiceCommon方法

  在该方法内判断ServiceConnection是否为null，然后通过ActivityManager.getService()获取ActivityManagerService，调用ActivityManagerService的bindService。

  ```
  private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
              handler, UserHandle user) {
          // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
          IServiceConnection sd;
          if (conn == null) {
              throw new IllegalArgumentException("connection is null");
          }
          if (mPackageInfo != null) {
              sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
          } else {
              throw new RuntimeException("Not supported in system context");
          }
          validateServiceIntent(service);
          try {
              IBinder token = getActivityToken();
              if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                      && mPackageInfo.getApplicationInfo().targetSdkVersion
                      < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                  flags |= BIND_WAIVE_PRIORITY;
              }
              service.prepareToLeaveProcess(this);
              int res = ActivityManager.getService().bindService(
                  mMainThread.getApplicationThread(), getActivityToken(), service,
                  service.resolveTypeIfNeeded(getContentResolver()),
                  sd, flags, getOpPackageName(), user.getIdentifier());
              if (res < 0) {
                  throw new SecurityException(
                          "Not allowed to bind to service " + service);
              }
              return res != 0;
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
  ```

- ActivityManagerService的bindService

  在该代码中内部调用了ActiveServices的bindServiceLocked方法。	

  ```
  public int bindService(IApplicationThread caller, IBinder token, Intent service,
              String resolvedType, IServiceConnection connection, int flags, String callingPackage,
              int userId) throws TransactionTooLargeException {
          enforceNotIsolatedCaller("bindService");
  
          // Refuse possible leaked file descriptors
          if (service != null && service.hasFileDescriptors() == true) {
              throw new IllegalArgumentException("File descriptors passed in Intent");
          }
  
          if (callingPackage == null) {
              throw new IllegalArgumentException("callingPackage cannot be null");
          }
  
          synchronized(this) {
              return mServices.bindServiceLocked(caller, token, service,
                      resolvedType, connection, flags, callingPackage, userId);
          }
      }
  ```

- ActiveServices的bindServiceLocked

  在该方法内又这样一段注释，当Service已经运行时我们可以直接连接，这段代码即是binderService的逻辑

  然后调用requestServiceBindingLocked方法。

  ```
  int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
              String resolvedType, final IServiceConnection connection, int flags,
              String callingPackage, final int userId) throws TransactionTooLargeException {
              ...
              if (s.app != null && b.intent.received) {
                  // Service is already running, so we can immediately
                  // publish the connection.
                  try {
                      c.conn.connected(s.name, b.intent.binder, false);
                  } catch (Exception e) {
                      Slog.w(TAG, "Failure sending service " + s.shortName
                              + " to connection " + c.conn.asBinder()
                              + " (in " + c.binding.client.processName + ")", e);
                  }
  
                  // If this is the first app connected back to this binding,
                  // and the service had previously asked to be told when
                  // rebound, then do so.
                  if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                      requestServiceBindingLocked(s, b.intent, callerFg, true);
                  }
              } else if (!b.intent.requested) {
                  requestServiceBindingLocked(s, b.intent, callerFg, false);
              }
              ...
              }
  ```

  requestServiceBindingLocked方法内r.app.thread.scheduleBindService，r.app.thread获取的是IApplicationThread，IApplicationThread是一个Binder类型的对象，它的具体实现是ActivityThread中的ApplicationThread，实际调用的是ActivityThread的内部类的ApplicationThread中的scheduleBindService

  ```
  private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
              boolean execInFg, boolean rebind) throws TransactionTooLargeException {
          if (r.app == null || r.app.thread == null) {
              // If service is not currently running, can't yet bind.
              return false;
          }
          if (DEBUG_SERVICE) Slog.d(TAG_SERVICE, "requestBind " + i + ": requested=" + i.requested
                  + " rebind=" + rebind);
          if ((!i.requested || rebind) && i.apps.size() > 0) {
              try {
                  bumpServiceExecutingLocked(r, execInFg, "bind");
                  r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                  r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                          r.app.repProcState);
                  if (!rebind) {
                      i.requested = true;
                  }
                  i.hasBound = true;
                  i.doRebind = false;
              } catch (TransactionTooLargeException e) {
                  // Keep the executeNesting count accurate.
                  if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r, e);
                  final boolean inDestroying = mDestroyingServices.contains(r);
                  serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                  throw e;
              } catch (RemoteException e) {
                  if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r);
                  // Keep the executeNesting count accurate.
                  final boolean inDestroying = mDestroyingServices.contains(r);
                  serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                  return false;
              }
          }
          return true;
      }
  ```

- ApplicationThread的scheduleBindService

  在该方法内最终调用sendMessage方法即最后调用ActivityThread内部的Handler的handleMessage方法

  ```
  public final void scheduleBindService(IBinder token, Intent intent,
                  boolean rebind, int processState) {
              updateProcessState(processState, false);
              BindServiceData s = new BindServiceData();
              s.token = token;
              s.intent = intent;
              s.rebind = rebind;
  
              if (DEBUG_SERVICE)
                  Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                          + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
              sendMessage(H.BIND_SERVICE, s);
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

  接着我们看mH（Handler）这个类中的handleMessage方法。由于上文传的类型是BIND_SERVICE然后看BIND_SERVICE下的代码，调用了handleBindService方法。

  ```
   ...
   case BIND_SERVICE:
                      Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                      handleBindService((BindServiceData)msg.obj);
                      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                      break;
    ....                  
  ```

  handleBindService方法内，会判断是否已经绑定过，如果未绑定调用Service的onBinder方法获取Binder对象，并且调用AMS的publishService，最终会在该方法内调用ServiceConnection的connected方法并将Binder对象传递过去。

  ```
   private void handleBindService(BindServiceData data) {
          Service s = mServices.get(data.token);
          if (DEBUG_SERVICE)
              Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
          if (s != null) {
              try {
                  data.intent.setExtrasClassLoader(s.getClassLoader());
                  data.intent.prepareToEnterProcess();
                  try {
                      if (!data.rebind) {
                          IBinder binder = s.onBind(data.intent);
                          ActivityManager.getService().publishService(
                                  data.token, data.intent, binder);
                      } else {
                          s.onRebind(data.intent);
                          ActivityManager.getService().serviceDoneExecuting(
                                  data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                      }
                      ensureJitEnabled();
                  } catch (RemoteException ex) {
                      throw ex.rethrowFromSystemServer();
                  }
              } catch (Exception e) {
                  if (!mInstrumentation.onException(s, e)) {
                      throw new RuntimeException(
                              "Unable to bind to service " + s
                              + " with " + data.intent + ": " + e.toString(), e);
                  }
              }
          }
      }
  ```








