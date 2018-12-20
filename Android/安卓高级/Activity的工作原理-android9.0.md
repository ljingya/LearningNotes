#### Activity的工作原理-android9.0

1. ###### startActivityForResult 方法

当开启一个页面的时候需要调用Activity的startActivity的方法。最终调用到了其内部的 startActivityForResult 方法。在 startActivityForResult 方法中 execStartActivity 方法的第二个参数mMainThread.getApplicationThread() ，mMainThread是 ActivityThread的实例，mMainThread.getApplicationThread() 返回 ApplicationThread ，ApplicationThread继承了  IApplicationThread.Stub，说明 ApplicationThread 是ActivityThread的一个内部的Binder 类。

```java
 public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            ...
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            ....
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

2. ###### 执行 Instrumentation 的 execStartActivity 的方法。

   ```
    public ActivityResult execStartActivity(
               Context who, IBinder contextThread, IBinder token, Activity target,
               Intent intent, int requestCode, Bundle options) {
           IApplicationThread whoThread = (IApplicationThread) contextThread;
           ...    
           try {
               intent.migrateExtraStreamToClipData();
               intent.prepareToLeaveProcess(who);
               //通过ActivityManager.getService()获取到ActivityManagerService（AMS），并执行其，内部的startActivity()
               int result = ActivityManager.getService()
                   .startActivity(whoThread, who.getBasePackageName(), intent,
                           intent.resolveTypeIfNeeded(who.getContentResolver()),
                           token, target != null ? target.mEmbeddedID : null,
                           requestCode, 0, null, options);
               checkStartActivityResult(result, intent);
           } catch (RemoteException e) {
               throw new RuntimeException("Failure from system", e);
           }
           return null;
       }
   ```

   在该方法中 ，启动Activity实际由ActivityManager.getService()返回的值启动，ActivityManager.getService()实际返回的 IActivityManager，IActivityManager是一个Binder对象，对应的实现ActivityManagerService。Singleton是一个封装的单例对象，当调用get方法时，会通过create方法获取IActivityManager。ActivityManager.getService()代码如下：

   ```
    	/**
        * @hide
        */
       public static IActivityManager getService() {
           return IActivityManagerSingleton.get();
       }
       
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
               new Singleton<IActivityManager>() {
                   @Override
                   protected IActivityManager create() {
                       final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                       final IActivityManager am = IActivityManager.Stub.asInterface(b);
                       return am;
                   }
               };   
   ```

   checkStartActivityResult 方法主要对结果进行校验。例如：当返回的结果是ActivityManager.START_INTENT_NOT_RESOLVED或者 ActivityManager.START_CLASS_NOT_FOUND时，则抛出常见的未在清单文件中注册的异常。实现如下：

   ```
   public static void checkStartActivityResult(int res, Object intent) {
           if (!ActivityManager.isStartResultFatalError(res)) {
               return;
           }
   
           switch (res) {
               case ActivityManager.START_INTENT_NOT_RESOLVED:
               case ActivityManager.START_CLASS_NOT_FOUND:
                   if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                       throw new ActivityNotFoundException(
                               "Unable to find explicit activity class "
                               + ((Intent)intent).getComponent().toShortString()
                               + "; have you declared this activity in your AndroidManifest.xml?");
                   throw new ActivityNotFoundException(
                           "No Activity found to handle " + intent);
               case ActivityManager.START_PERMISSION_DENIED:
                   throw new SecurityException("Not allowed to start activity "
                           + intent);
               case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
                   throw new AndroidRuntimeException(
                           "FORWARD_RESULT_FLAG used while also requesting a result");
               case ActivityManager.START_NOT_ACTIVITY:
                   throw new IllegalArgumentException(
                           "PendingIntent is not an activity");
               case ActivityManager.START_NOT_VOICE_COMPATIBLE:
                   throw new SecurityException(
                           "Starting under voice control not allowed for: " + intent);
               case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
                   throw new IllegalStateException(
                           "Session calling startVoiceActivity does not match active session");
               case ActivityManager.START_VOICE_HIDDEN_SESSION:
                   throw new IllegalStateException(
                           "Cannot start voice activity on a hidden session");
               case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
                   throw new IllegalStateException(
                           "Session calling startAssistantActivity does not match active session");
               case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
                   throw new IllegalStateException(
                           "Cannot start assistant activity on a hidden session");
               case ActivityManager.START_CANCELED:
                   throw new AndroidRuntimeException("Activity could not be started for "
                           + intent);
               default:
                   throw new AndroidRuntimeException("Unknown error code "
                           + res + " when starting " + intent);
           }
       }
   ```

3. ActivityManagerService的startActivity

   执行ActivityManagerService中的startActivity，最终会调用 startActivityAsCaller，在这个方法内，通过ActivityStartController的 obtainStarter方法获取到 ActivityStarter,用过ActivityStarter设置执行需要的值，代码如下：

   ```
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
               boolean validateIncomingUser) {
   	......
           // TODO: Switch to user app stacks here.
           return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                   .setCaller(caller)
                   .setCallingPackage(callingPackage)
                   .setResolvedType(resolvedType)
                   .setResultTo(resultTo)
                   .setResultWho(resultWho)
                   .setRequestCode(requestCode)
                   .setStartFlags(startFlags)
                   .setProfilerInfo(profilerInfo)
                   .setActivityOptions(bOptions)
                   .setMayWait(userId)
                   .execute();
   
       }
   ```

   ActivityStarter最后执行execute()方法，在ActivityStarter调用setMayWait时会将mayWait设置为true。

   setMayWait方法内部实现：

   ```
   ActivityStarter setMayWait(int userId) {
           mRequest.mayWait = true;
           mRequest.userId = userId;
           return this;
       }
   ```

   execute()方法内部实现：

   ```
   int execute() {
           try {
               // TODO(b/64750076): Look into passing request directly to these methods to allow
               // for transactional diffs and preprocessing.
               if (mRequest.mayWait) {
                   return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                           mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                           mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                           mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                           mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                           mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                           mRequest.inTask, mRequest.reason,
                           mRequest.allowPendingRemoteAnimationRegistryLookup);
               } else {
                   return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                           mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                           mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                           mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                           mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                           mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                           mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                           mRequest.outActivity, mRequest.inTask, mRequest.reason,
                           mRequest.allowPendingRemoteAnimationRegistryLookup);
               }
           } finally {
               onExecutionComplete();
           }
       }
   ```

   接着调用startActivityMayWait方法直到调用到内部的startActivity方法,接着调用到 \startActivityUnchecked方法 ，startActivity方法代码如下：

   ```
   private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                   IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                   int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                   ActivityRecord[] outActivity) {
           int result = START_CANCELED;
           try {
               mService.mWindowManager.deferSurfaceLayout();
                //调用到 startActivityUnchecked
               result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                       startFlags, doResume, options, inTask, outActivity);
           } finally {
          ....
    
           }
   
           postStartActivityProcessing(r, result, mTargetStack);
   
           return result;
       }
   ```

   startActivityUnchecked方法内部，通过启动的标志位及启动模式决定一个Activity的启动，最终都会调用 ActivityStackSupervisor的resumeFocusedStackTopActivityLocked()，startActivityUnchecked方法如下：

   ```
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
               IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
               int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
               ActivityRecord[] outActivity) {
   
         ....
   //通过启动的标志位及启动模式决定一个Activity的启动。
           final boolean dontStart = top != null && mStartActivity.resultTo == null
                   && top.realActivity.equals(mStartActivity.realActivity)
                   && top.userId == mStartActivity.userId
                   && top.app != null && top.app.thread != null
                   && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                   || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK));
           if (dontStart) {
               // For paranoia, make sure we have correctly resumed the top activity.
               topStack.mLastPausedActivity = null;
               if (mDoResume) {
               
                   mSupervisor.resumeFocusedStackTopActivityLocked();
               }
               ....
   
               deliverNewIntent(top);
   
   
               return START_DELIVERED_TO_TOP;
           }
   
   
           if (mDoResume) {
               final ActivityRecord topTaskActivity =
                       mStartActivity.getTask().topRunningActivityLocked();
               if (!mTargetStack.isFocusable()
                       || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                       && mStartActivity != topTaskActivity)) {
                 
                   mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                           mOptions);
               }
           } else if (mStartActivity != null) {
               mSupervisor.mRecentTasks.add(mStartActivity.getTask());
           }
          
   
           return START_SUCCESS;
       }
   ```

4. ###### ActivityStackSupervisor的resumeFocusedStackTopActivityLocked()

   resumeFocusedStackTopActivityLocked方法内部最终会调用ActivityStack的resumeTopActivityUncheckedLocked（）方法。

   ```
   
       boolean resumeFocusedStackTopActivityLocked() {
           return resumeFocusedStackTopActivityLocked(null, null, null);
       }
   
       boolean resumeFocusedStackTopActivityLocked(
               ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
   
           if (!readyToResume()) {
               return false;
           }
   
           if (targetStack != null && isFocusedStack(targetStack)) {
               return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
           }
   
           final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
           if (r == null || !r.isState(RESUMED)) {
               mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
           } else if (r.isState(RESUMED)) {
               // Kick off any lingering app transitions form the MoveTaskToFront operation.
               mFocusedStack.executeAppTransition(targetOptions);
           }
   
           return false;
       }
   ```

5. ActivityStack的resumeTopActivityUncheckedLocked（）

   resumeTopActivityUncheckedLocked方法内部调用了resumeTopActivityInnerLocked，在resumeTopActivityInnerLocked方法内部会判断是否有处在onResume状态的Activity,若有先调用startPausingLocked改变Activity的状态。接着调用ActivityStackSupervisor 的startSpecificActivityLocked方法

   ```
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    
   if (mResumedActivity != null) {
               if (DEBUG_STATES) Slog.d(TAG_STATES,
                       "resumeTopActivityLocked: Pausing " + mResumedActivity);
               pausing |= startPausingLocked(userLeaving, false, next, false);
           }
           ....
            mStackSupervisor.startSpecificActivityLocked(next, true, false);
       }
   ```

   ##### Activity的暂停

6. 分析startPausingLocked方法

   在 startPausingLocked方法内部 调用AMS的 getLifecycleManager()获取到ClientLifecycleManager，该类是客户端生命周期的管理类，接着调用ClientLifecycleManager的 scheduleTransaction方法，该方法的第二个参数 传递了PauseActivityItem，该参数是Activity对应的onPause状态的类，继承自 ActivityLifecycleItem。

   ```
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
               ActivityRecord resuming, boolean pauseImmediately) {
        ....
         if (prev.app != null && prev.app.thread != null) {
               if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
               try {
                   EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                           prev.shortComponentName, "userLeaving=" + userLeaving);
                   mService.updateUsageStats(prev, false);
   
                   mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                           PauseActivityItem.obtain(prev.finishing, userLeaving,
                                   prev.configChangeFlags, pauseImmediately));
               } catch (Exception e) {
                   // Ignore exception, if process died other code will cleanup.
                   Slog.w(TAG, "Exception thrown during pause", e);
                   mPausingActivity = null;
                   mLastPausedActivity = null;
                   mLastNoHistoryActivity = null;
               }
           } else {
               mPausingActivity = null;
               mLastPausedActivity = null;
               mLastNoHistoryActivity = null;
           }
         .......
         }      
   ```

7. ClientLifecycleManager的scheduleTransaction方法

   该方法调用 最终调用 ClientTransaction的schedule方法。

   ```
   void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
               @NonNull ActivityLifecycleItem stateRequest) throws RemoteException {
           final ClientTransaction clientTransaction = transactionWithState(client, activityToken,
                   stateRequest);
           scheduleTransaction(clientTransaction);
       }
       
     
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
           final IApplicationThread client = transaction.getClient();
           transaction.schedule();
           if (!(client instanceof Binder)) {
               // If client is not an instance of Binder - it's a remote call and at this point it is
               // safe to recycle the object. All objects used for local calls will be recycled after
               // the transaction is executed on client in ActivityThread.
               transaction.recycle();
           }
       }    
   ```

8. ClientTransaction的schedule

   该方法中，mClient是IApplicationThread的对象，由于 IApplicationThread是一个Binder类型的对象，它的实现是ActivityThread中的 ApplicationThread，最终调用的是 ApplicationThread的 scheduleTransaction

   ```
   public void schedule() throws RemoteException {
           mClient.scheduleTransaction(this);
       }
   ```

9. ApplicationThread的 scheduleTransaction

   该方法中，调用ActivityThread的scheduleTransaction，由于ActivityThread中并未实现该方法因此，具体实现在其ClientTransactionHandler类中

   ```
   @Override
           public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
               ActivityThread.this.scheduleTransaction(transaction);
           }
   ```

10. ClientTransactionHandler的scheduleTransaction

    在该方法调用sendMessage方法，由于sendMessage是抽象的所以具体实现是在ActivityThread中

    ```
    void scheduleTransaction(ClientTransaction transaction) {
            transaction.preExecute(this);
            sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
        }
    ```

11. ActivityThread的sendMessage

    在实现的sendMessage中，最终通过mH即内部的H继承的Handler发送消息，最终调用handleMessage方法。

    ```
     void sendMessage(int what, Object obj) {
            sendMessage(what, obj, 0, 0, false);
        }
    
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

    在handleMessage中有这样一段处理代码,调用TransactionExecutor的execute处理消息。

    ```
    case EXECUTE_TRANSACTION:
                        final ClientTransaction transaction = (ClientTransaction) msg.obj;
                        mTransactionExecutor.execute(transaction);
                        if (isSystem()) {
                            // Client transactions inside system process are recycled on the client side
                            // instead of ClientLifecycleManager to avoid being cleared before this
                            // message is handled.
                            transaction.recycle();
                        }
                        // TODO(lifecycler): Recycle locally scheduled transactions.
                        break;
    ```

12. TransactionExecutor的execute

    该方法中调用executeCallbacks和executeLifecycleState方法。

    ```
         */
        public void execute(ClientTransaction transaction) {
            final IBinder token = transaction.getActivityToken();
            log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);
    
            executeCallbacks(transaction);
    
            executeLifecycleState(transaction);
            mPendingActions.clear();
            log("End resolving transaction");
        }
    ```

    executeCallBacks方法：由于未设置callbacks，直接返回。

    ```
    public void executeCallbacks(ClientTransaction transaction) {
            final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
            if (callbacks == null) {
                // No callbacks to execute, return early.
                return;
            }
       .....
        }
    ```

    executeLifecycleState方法：

    该方法中调用 lifecycleItem.execute方法，lifecycleItem是PauseActivityItem的实例，因此直接查看PauseActivityItem中的execute方法。

    ```
        /** Transition to the final state if requested by the transaction. */
        private void executeLifecycleState(ClientTransaction transaction) {
            final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
            if (lifecycleItem == null) {
                // No lifecycle request, return early.
                return;
            }
            log("Resolving lifecycle state: " + lifecycleItem);
    
            final IBinder token = transaction.getActivityToken();
            final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
    
            if (r == null) {
                // Ignore requests for non-existent client records for now.
                return;
            }
    
            // Cycle to the state right before the final requested state.
            cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);
    
            // Execute the final transition with proper parameters.
            lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
            lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
        }
    ```

13. PauseActivityItem中的execute方法

    该方法中调用   client.handlePauseActivity方法，client是ClientTransactionHandler。由于ClientTransactionHandler中的handlePauseActivity是一个抽象方法，因此需要查看ActivityThread的handlePauseActivity。

    ```
      @Override
        public void execute(ClientTransactionHandler client, IBinder token,
                PendingTransactionActions pendingActions) {
            Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
            client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                    "PAUSE_ACTIVITY_ITEM");
            Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
        }
    ```

14. ActivityThread的handlePauseActivity

    在该方法中调用了performPauseActivity方法。然后调用performPauseActivityIfNeeded方法，在performPauseActivityIfNeeded方法中调用了mInstrumentation.callActivityOnPause(r.activity)的方法。

    ```
      @Override
        public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
                int configChanges, PendingTransactionActions pendingActions, String reason) {
            ActivityClientRecord r = mActivities.get(token);
            if (r != null) {
                if (userLeaving) {
                    performUserLeavingActivity(r);
                }
    
                r.activity.mConfigChangeFlags |= configChanges;
                performPauseActivity(r, finished, reason, pendingActions);
    
                // Make sure any pending writes are now committed.
                if (r.isPreHoneycomb()) {
                    QueuedWork.waitToFinish();
                }
                mSomeActivitiesChanged = true;
            }
        }
    ```

15. Instrumentation的callActivityOnPaus

    最终调用activity.performPause();

    ```
     public void callActivityOnPause(Activity activity) {
            activity.performPause();
        }
    ```

16. Activity的performPause

    在performPause方法中最终调用了 onPause()方法。

    ```
     final void performPause() {
     ....
            onPause();
           ...
        }
    ```

    到这里，当启动一个Activity时，上一个activity的从onResume状态编程onpasue状态分析完毕。接下来分析Activity的启动。

##### 应用已经启动的条件下Activity的启动。

17. ActivityStackSupervisor 的startSpecificActivityLocked


##### 应用未启动的条件下Activity的启动。

