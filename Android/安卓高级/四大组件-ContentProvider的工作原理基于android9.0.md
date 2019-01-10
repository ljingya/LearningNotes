##### 四大组件-ContentProvider的工作原理基于android9.0

ContentProvider是安卓的四大组件之一，底层使用Binder，可以用于跨进程通信，另外ContentProvider的启动伴随着进程的启动，进程的启动调用Process的start方法，并且新进程的入口是ActivityThread的main方法。接下来分析ContentProvider如何启动。

1. ###### ContentProvider的启动

   在main方法中，会调用ActivityManagerService中的attachApplication方法。然后调用attachApplicationLocked方法，在该方法中调用IApplicationThread的bindApplication绑定Application方法，IApplicationThread的Binder对象的实现类是ActivityThread内部的ApplicationThread。

   ```
    @Override
       public final void attachApplication(IApplicationThread thread, long startSeq) {
           synchronized (this) {
               int callingPid = Binder.getCallingPid();
               final int callingUid = Binder.getCallingUid();
               final long origId = Binder.clearCallingIdentity();
               attachApplicationLocked(thread, callingPid, callingUid, startSeq);
               Binder.restoreCallingIdentity(origId);
           }
       }
   ```

   ```
    private final boolean attachApplicationLocked(IApplicationThread thread,
               int pid, int callingUid, long startSeq) {
        ....
   thread.bindApplication(processName, appInfo, providers,
                           app.instr.mClass,
                           profilerInfo, app.instr.mArguments,
                           app.instr.mWatcher,
                           app.instr.mUiAutomationConnection, testMode,
                           mBinderTransactionTrackingEnabled, enableTrackAllocation,
                           isRestrictedBackupMode || !normalMode, app.persistent,
                           new Configuration(getGlobalConfiguration()), app.compat,
                           getCommonServicesLocked(app.isolated),
                           mCoreSettingsObserver.getCoreSettingsLocked(),
                           buildSerial, isAutofillCompatEnabled);
       .....                    
                           }
   ```

   ApplicationThread的bindApplication方法，在该方法中会调用 sendMessage(H.BIND_APPLICATION, data) 方法，H是Activity的内部的Handler，最终调用Handler的sendMessage方法。最终调用Handler的handleMessage方法。Handler内部状态为BIND_APPLICATION的处理方法。调用handleBindApplication方法

   ```
    case BIND_APPLICATION:
                       Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                       AppBindData data = (AppBindData)msg.obj;
                       handleBindApplication(data);
                       Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                       break;
   ```

   handleBindApplication方法，在该方法内部installContentProviders这句代码就是启动ContentProvider的方法，启动之后调用callApplicationOnCreate方法，这句代码是调用了Application的onCreate方法。所以得出一个结论ContentProvider的启动在调用Application的onCreate方法之前。

   ```
    	....
    // don't bring up providers in restricted mode; they may depend on the
               // app's custom Application class
               if (!data.restrictedBackupMode) {
                   if (!ArrayUtils.isEmpty(data.providers)) {
                       installContentProviders(app, data.providers);
                       // For process that contains content providers, we want to
                       // ensure that the JIT is enabled "at some point".
                       mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                   }
               }
   
         .....
                mInstrumentation.callApplicationOnCreate(app);
         ......
   ```

   installContentProviders的方法，在该方法内部调用installProvider方法。接着调用localProvider即ContentProvider方法的attachInfo方法。

   ```
   rivate ContentProviderHolder installProvider(Context context,
               ContentProviderHolder holder, ProviderInfo info,
               boolean noisy, boolean noReleaseNeeded, boolean stable) {
               ...
                // XXX Need to create the correct context for this provider.
                   localProvider.attachInfo(c, info);
               ...
               }
   ```

   attachInfo方法内部中调用了ContentProvider的onCreate方法，到这里ContentProvider的启动就结束了。

   ```
      private void attachInfo(Context context, ProviderInfo info, boolean testing) {
           mNoPerms = testing;
   
           /*
            * Only allow it to be set once, so after the content service gives
            * this to us clients can't change it.
            */
           if (mContext == null) {
               mContext = context;
               if (context != null) {
                   mTransport.mAppOpsManager = (AppOpsManager) context.getSystemService(
                           Context.APP_OPS_SERVICE);
               }
               mMyUid = Process.myUid();
               if (info != null) {
                   setReadPermission(info.readPermission);
                   setWritePermission(info.writePermission);
                   setPathPermissions(info.pathPermissions);
                   mExported = info.exported;
                   mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
                   setAuthorities(info.authority);
               }
               ContentProvider.this.onCreate();
           }
       }
   ```
