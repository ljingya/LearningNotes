### Service基础

1. ##### 生命周期

   startService方式：onCreate()—>onStartCommand()—>onDestory();

   bindService方式：onCreate()—>onBind—>onUnbind()—>onDestory()

2. ##### startForeground()

    

3. ##### onStartCommand()与START_STICKY

   START_STICKY：该值是onStartCommand()的返回值,当服务所在的进程被杀死，服务处于启动状态。随后系统会重建Service，但是不保留intent的状态，由于服务是开始的会重新调用onStartCommand()方法，如果后续没有开始命令发送到Service，那么Intent为null。

   START_NOT_STICKY：使用这歌返回值时，服务被异常杀掉，不会重启。

   START_REDELIVER_INTENT：当服务被异常kill时，系统会重启服务，并将最近一次的Intent的值传入。

4. ##### priority

   ```
    <service android:name=".chapter01.service.ServiceKg">
         <intent-filter android:priority="1000">
         //必须添加action
           <action android:name="android.intent.action.service"/>
         </intent-filter>
       </service>
   ```

   Service可以通过设置priority提升Service的优先级，防止被系统回收，1000时最大值，priority的值越小，优先级越低

   