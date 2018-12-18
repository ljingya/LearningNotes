### Activity的基础知识

------



1. ##### 思维导图

   ![Activity基础知识](https://raw.githubusercontent.com/ljingya/LearningNotes/master/Android/Activity/Image/Activity%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86.jpg)

2. ##### Activity的生命周期

   ![Activity的生命周期](https://raw.githubusercontent.com/ljingya/LearningNotes/master/Android/Activity/Image/Activity%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.jpg)

3. ##### 启动模式

   Activity的启动模式需要在清单文件中的activity标签中设置：

   ```
   android:launchMode="*"
   ```

   *代表下面的四种启动模式，当不设置时，默认是Standard。

   - ###### Standard

     Standard：标准启动模式，也是默认的启动模式，使用该模式有以下特点，启动Activity时会创建该Activity的实例，并且启动的Activity与启动该Activity的页面在同一个栈中。另外当启动Activity时，启动的Activity并不会调用onNewIntent方法。

   - ###### SingleTop

     SingleTop：栈顶复用模式。列入A  B两个页面，B页面的启动模式设置为 SingleTop，A启动B，然后B再启动一个B页面，此时B页面位于栈顶 ，B页面不会走 onCreate 的生命周期,而是调用 onNewIntent方法。然后在栈顶的B页面启动A页面，由于A页面不在栈顶，所以A页面需要重新创建调用完整的生命周期

   - ###### SingleTask

     SingleTask：栈内复用模式。当启动一个页面，会从占中查找是否存在该页面的实例，若存在，将该页面调到栈顶，并调用onNewIntent方法。若不存在，则创建该页面的实例，压栈。

   - ###### SingleInstance

     SingleInstance：单实例模式，A ，B两个页面，B设置为 SingleInstance 模式时，当A启动 B时，会为B重新创建一个栈，而且这个栈中只有B页面的一个实例，后续再启动 B，不会在栈中创建 B 的实例。而且后续启动B 页面，只会调用B页面的 onNewIntent方法。

4. ##### onNewIntent

   当页面的启动模式为Single Top 时，当启动的页面已经位于栈顶时，会调用该页面的 onNewIntent -> onRestart ->onStart ->onResume 方法。

   当页面的启动模式为 SingleTask时。当启动的页面已经位于栈内，会调用该页面的 onNewIntent -> onRestart ->onStart ->onResume 方法。

   当页面的启动模式为 SingleInstance时，当启动的页面已经单独位于一个栈中时，后续再启动该页面，会调用该页面的 onNewIntent -> onRestart ->onStart ->onResume 方法。

5. ##### 页面恢复

   当页面由于配置文件导致页面意外死亡，如横竖屏切换，该页面会调用 onCreate-> onSaveInstanceState-> onDestroy->onCreate->onRestoreInstanceState

   - ###### onSaveInstanceState

     在该方法中将要保存的数据存储在Bundle中。

   - ###### onRestoreInstanceState

     在该方法中将之前保存的Bundle中的数据取出。