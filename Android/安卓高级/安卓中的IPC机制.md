

#### IPC简介
IPC ：Inter-Process Communication的简写，叫进程间通信，指的是两个进程间通信，何为进程？进程与线程的区别？IPC的使用场景？

##### 何为进程
进程指的是一个可执行的单元，在PC和移动设备是一个程序和应用。
##### 进程与线程的区别

线程是cpu可调度的最小单元，同时线程是一种有限制的资源，一个进程可以包含多个线程或者单个线程，当包含单个线程时，该线程就是主线程。
##### IPC的使用场景

1. 应用自身由于某些原因需要采用多进程，如某些模块由于特殊原因需要放在单独的进程中，另外为了增加应用的内存。

1. 当前应用需要向另一个应用提供数据。由于是两个进程，需要采用跨进程的通信。

#### 多进程模式

那么如何开启多进程？多进程的运行机制是什么呢？

##### 如何开启多进程
```
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity">
      <intent-filter>
        <action android:name="android.intent.action.MAIN"/>

        <category android:name="android.intent.category.LAUNCHER"/>
      </intent-filter>
    </activity>
    <activity android:name=".SecondActivity"
      android:process=":remote"
      />
    <activity android:name=".ThirdActivity"
      android:process="com.example.lijingya.androidipc.remote"
      />
  </application>
```
在android中使用多进程只有一种方法，就是对四大组件指定 android:process 属性，另外你无法对一个线程或者一个实体类单独指定一个进程。代码中使用  android:process=":remote" 和  android:process="com.example.lijingya.androidipc.remote" 分别为 SecondActivity 和 ThirdActivity 单独指定了一个进程。 运行之后

![image](https://raw.githubusercontent.com/JyBeyond/DemoImage/master/Binder/binder-1.jpg)

如图显示了三个进程。

android:process=":remote" 与 android:process="com.example.lijingya.androidipc.remote"的区别？

 主要有两点：以“：”开头的在前面会附加应用的包名，而ThirdActivity的命名方式是一种完整的命名方式，不会附加应用的包名。
 
 其次以“：”开头的命名方式是应用的私有方式，其他应用的组件不可以和它在同一个进程中，而另外一种可通过ShareUId方式和她跑在同一个进程中。
 
 关于UID和PID可以看这篇介绍
 
 [UID与PID](https://www.cnblogs.com/perseus/articles/2354173.html)

##### 多进程的工作机制

开启多进程会导致以下问题：
1. 静态变量和单例模式失效
2. 同步失效
3. SharePrefrence 的可靠性下降
4. Applciation多次创建。

对1来说，安卓为每个应用程序分配了一个虚拟机和一块内存空间或者这样说为每个进程分配了一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，每个虚拟机访问的都是同一个类的对象的副本。由于同步需要获取同一个对象的锁，既然都已经不是同一个对象了，肯定获取到同一个类的对象锁也不一样。第三个问题， SharePrefrence 底层是Xml读写，无法实现同步，肯定不能实现并发的读写。第四个问题，当一个组件在新的进程中时，程序会要创建新的进程和分配虚拟机，所以就是一个启动应用的过程，所以Application的oncreate方法会调用多次。


#### IPC基础介绍

##### Serializable接口

Serializable 是一个数据序列化的接口，是一个空实现
```
public interface Serializable {
}

```
想让一个类实现序列化需要实现 Serializable 接口，然后指定 serialVersionUID，当然在 serialVersionUID 也不是必须的，可以不用指定也可以实现序列化，但是当反序列化的时候如果没有指定，当数据发生变化时，会发生序列化失败。

serialVersionUID的工作机制：序列化的时候，系统会把当前类的serialVersionUID写入到文件中，当反序列化的时候会去检测文件中的id，和类中的id是否一致，如果一致，说明当前版本和序列化的版本一致。可以成功反序列化，否则就说明当前类发生了某些变化。因此当手动指定或者用ecplise或者ide自动生成时，当不是数据结构发生非常规性变化（修改类名，修改成员变量的类型等）时，反序列化依然可以成功。

```
public class UserManager implements Serializable {

    private static final long serialVersionUID = -2066517753597756018L;
    
    public static int uid = 1;

}
```




##### Parcelable 接口
Parcelable是安卓中的一种序列化方式，需要类实现Parcelable，
```
public class User implements Parcelable {

    private String name;
    private int age;
    private VipUser mVipUser;

    /**
     *从序列化后的对象中创建原始对象
     * @param in 内部包装了可序列化的数据，可以在Binder中自由传输。
     */
    protected User(Parcel in) {
        name = in.readString();
        age = in.readInt();
        //VipUser是一个可序列化的对象，因此需要传上下文的类加载器
        mVipUser = in.readParcelable(VipUser.class.getClassLoader());
    }

    /**
     * 序列化功能
     * 将当前对象写入序列化中，
     * @param dest
     * @param flags
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(age);
        dest.writeParcelable(mVipUser, flags);
    }

    /**
     * 内容描述功能
     * 返回当前对象的内容描述，如果含有文件描述符，返回1，否则返回0，几乎所有都返回0.
     * @return
     */
    @Override
    public int describeContents() {
        return 0;
    }

    /**
     * 反序列化功能
     */
    public static final Creator<User> CREATOR = new Creator<User>() {
        /**
         * 从序列化后的对象中创建原始对象
         * @param in
         * @return
         */
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }

        /**
         * 创建指定对象的原始对象数组
         * @param size
         * @return
         */
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
}
```
##### Parceable与Serializable区别

两者都可以用于序列化和Intent之间传递数据，Serializable是java的序列化接口，使用简单但开销大，Parceable是Android 平台提供的序列化接口，缺点是效率高，但是使用麻烦，Parceable主要用在内存序列化上，将对象序列化到设备存储中，或将对象序列化后进行网络存储可以使用Serializable。

##### Binder 
Binder是安卓中的类，实现了IBinder接口，Binder是Android中的一种跨进程通信方式，Binder从FrameWork的角度看，是连接Servivemanager与Manager之间的桥梁，从应用层看角度说，可以实现客户端与服务端的通信，客户端通过binderService可以获取服务端的Binder对象，调用服务端的服务和方法。Android开发中，Bidner主要用在Service中，Binder包括AIDL和Messenger ，而Messenger的底层也是AIDL。接下来就以AIDL说明BinDer的工作机制。

新建Book类实现Parcelable。
```
public class Book implements Parcelable {

    private int bookId;
    private String bookName;

    protected Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
}
```
创建Book.aidl和IBookManager.aidl
```
package com.example.lijingya.androidipc.aidl;

parcelable Book;
```

```
import com.example.lijingya.androidipc.aidl.Book;
interface IBookManager {
  List<Book> getBookList();
  void addBook(in Book book);
}
```
IBookManager.aidl中有一个获取书籍的方法和添加书籍的方法。当编译时会在build/genrated/source/aidl中下生成IBookManager。IBookManager源码如下：
```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: D:\\demo_work\\AndroidIpc\\app\\src\\main\\aidl\\com\\example\\lijingya\\androidipc\\aidl\\IBookManager.aidl
 */
package com.example.lijingya.androidipc.aidl;

/**
 * IBookManager继承IInterface接口
 */
public interface IBookManager extends android.os.IInterface {

    /**
     * Local-side IPC implementation stub class.
     * Stub是Binder，当客户端和服务端在同一个进程时，不会走transact方法，
     * 当他们不再同一个进程时，方法调用需要走transact过程，这个逻辑由内部的Proxy代理类完成
     */
    public static abstract class Stub extends android.os.Binder implements com.example.lijingya.androidipc.aidl.IBookManager {

        /**
         * Binder的唯一标识。一般是当前类名
         */
        private static final java.lang.String DESCRIPTOR = "com.example.lijingya.androidipc.aidl.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.lijingya.androidipc.aidl.IBookManager interface,
         * generating a proxy if needed.
         *用于将服务端的Binder对象转化成客户端所需要的AIDL对象，当服务端和客户端在同一个进程中时，返回本身，当不在同一个进程中时，
         * 则通过Proxy代理类实现。
         */
        public static com.example.lijingya.androidipc.aidl.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }

            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.lijingya.androidipc.aidl.IBookManager))) {
                return ((com.example.lijingya.androidipc.aidl.IBookManager) iin);
            }
            return new com.example.lijingya.androidipc.aidl.IBookManager.Stub.Proxy(obj);
        }

        /**
         * 返回客户端需要的BInder对象
         * @return
         */
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
         * 运行在服务端的BInder线程池中，客户端发起请求时，会交由此方法处理
         * @param code   标识需要调用的方法
         * @param data   从data中取出目标方法的参数，然后执行目标方法
         * @param reply  若目标方法有返回值，就向reply中回写数据
         * @param flags
         * @return
         * @throws android.os.RemoteException
         */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.example.lijingya.androidipc.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.lijingya.androidipc.aidl.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.lijingya.androidipc.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.lijingya.androidipc.aidl.IBookManager {

            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * 运行在客户端
             * @return
             * @throws android.os.RemoteException
             */
            @Override
            public java.util.List<com.example.lijingya.androidipc.aidl.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.lijingya.androidipc.aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.lijingya.androidipc.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            /**
             * 运行在客户端
             * @param book
             * @throws android.os.RemoteException
             */
            @Override
            public void addBook(com.example.lijingya.androidipc.aidl.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        /**
         * 客户端发起请求时，这两个Id用于标识在transact过程中客户端请求的方法。
          */
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    /**
     * IBookManager.aidl中声明的方法
     * @return
     * @throws android.os.RemoteException
     */
    public java.util.List<com.example.lijingya.androidipc.aidl.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.example.lijingya.androidipc.aidl.Book book) throws android.os.RemoteException;
}

```
上面便是Binder的工作机制，有两点需注意，
1. 当客户端发起请求时，便会挂起，知道服务端返回数据。所以发起请求不能在UI线程中。
2. Binder的方法运行在Binder的线程池中，所以Binder的方法不管是否耗时都应该采用同步的方法。
![image](https://raw.githubusercontent.com/JyBeyond/DemoImage/master/Binder/binder-2.jpg)

其实AIDL只是提供了我们简便书写Binder的方法，我们也可以自己手写一个Binder。

##### 实现一个 IBookManager 接口
```
public interface IBookManager extends IInterface{

    /**
     * Binder的唯一标识。一般是当前类名
     */
    static final java.lang.String DESCRIPTOR = "com.example.lijingya.androidipc.manualbinder.IBookManager";

    /**
     * 客户端发起请求时，这两个Id用于标识在transact过程中客户端请求的方法。
     */
    static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);

    static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

    void addBook(Book book) throws RemoteException;

    List<Book> getBookList() throws RemoteException;
}
```
##### 实现一个类似Stub的内部类 IBookManagerImpl
```
public class IBookManagerImpl extends Binder implements IBookManager {


    public IBookManagerImpl() {
        this.attachInterface(this, DESCRIPTOR);
    }


    public static IBookManager asInterface(IBinder obj) {
        if ((obj == null)) {
            return null;
        }

        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IBookManager))) {
            return ((IBookManager) iin);
        }
        return new IBookManagerImpl.Proxy(obj);
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_getBookList: {
                data.enforceInterface(DESCRIPTOR);
                java.util.List<com.example.lijingya.androidipc.aidl.Book> _result = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(_result);
                return true;
            }
            case TRANSACTION_addBook: {
                data.enforceInterface(DESCRIPTOR);
                com.example.lijingya.androidipc.aidl.Book _arg0;
                if ((0 != data.readInt())) {
                    _arg0 = com.example.lijingya.androidipc.aidl.Book.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
                this.addBook(_arg0);
                reply.writeNoException();
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }

    private static class Proxy implements IBookManager {

        private android.os.IBinder mRemote;

        Proxy(android.os.IBinder remote) {
            mRemote = remote;
        }

        @Override
        public android.os.IBinder asBinder() {
            return mRemote;
        }

        public java.lang.String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }

        /**
         * 运行在客户端
         * @return
         * @throws android.os.RemoteException
         */
        @Override
        public java.util.List<com.example.lijingya.androidipc.aidl.Book> getBookList() throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            java.util.List<com.example.lijingya.androidipc.aidl.Book> _result;
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                mRemote.transact(TRANSACTION_getBookList, _data, _reply, 0);
                _reply.readException();
                _result = _reply.createTypedArrayList(com.example.lijingya.androidipc.aidl.Book.CREATOR);
            } finally {
                _reply.recycle();
                _data.recycle();
            }
            return _result;
        }

        /**
         * 运行在客户端
         * @param book
         * @throws android.os.RemoteException
         */
        @Override
        public void addBook(com.example.lijingya.androidipc.aidl.Book book) throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                if ((book != null)) {
                    _data.writeInt(1);
                    book.writeToParcel(_data, 0);
                } else {
                    _data.writeInt(0);
                }
                mRemote.transact(TRANSACTION_addBook, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
        }
    }



    @Override
    public void addBook(Book book) throws RemoteException {

    }

    @Override
    public List<Book> getBookList() throws RemoteException {
        return null;
    }

    @Override
    public IBinder asBinder() {
        return this;
    }
}
```
手写的和自动实现的基本一样。只是将结构进行调整。

#### Android中IPC方式

##### AIDL

1. ##### 定义AIDL接口
关于如何定义AIDL接口已经在binder章节说明过，这里就不过多讲述。

2. ##### 远程服务端Service的实现
在安卓中创建服务端主要使用Service组件，在这里我们使用CopyOnWriteArrayList这个线程同步的类，这是java提供的一个线程同步类的。之前在binder章节说过，BInder的方法是执行在BInder线程池中，因此需要处理线程同步，在这里直接使用CopyOnWriteArrayList自同步。
```
public class BookService extends Service {

    private CopyOnWriteArrayList<Book> mBooks = new CopyOnWriteArrayList<>();

    private IBinder mIBinder = new Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {

            return mBooks;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBooks.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        mBooks.add(new Book(1, "lijingya1"));
        mBooks.add(new Book(2, "lijingya2"));
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mIBinder;
    }
}
```

然后需要在清单文件注册并设置多进程。
```
 <service
    android:name=".aidl.BookService"
    android:process="com.example.lijingya.androidipc.aidl.remote"
/>
```

3. ##### 客户端的实现
客户端主要是进行绑定，并获取远程服务端Binder的应用，并调用内部的方法。 

```
public class BookActivity extends AppCompatActivity implements OnClickListener {

    private static final String TAG = "BookActivity";

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager iBookManager = Stub.asInterface(service);
            try {
                List<Book> bookList = iBookManager.getBookList();
                Log.d(TAG, "查询 书籍 ：" + bookList.toString());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book);
        Button btnStartRemoteService = findViewById(R.id.btn_start_remote_service);
        btnStartRemoteService.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_start_remote_service:
                Intent intent = new Intent(this, BookService.class);
                bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
                break;
            default:
                break;
        }
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();
    }
}
```
4. ##### 监听远程服务端开。
DeathRecipient类是一个接口，内部只有binderDied方法，当Binder死亡时，会回调binderDied方法。

```
 /**
     * 监听远程服务端意外死亡，重连
     */
    private IBinder.DeathRecipient mDeathRecipient = new DeathRecipient() {
        @Override
        public void binderDied() {
            if (iBookManager == null) {
                return;
            }
            Log.d(TAG, "Service Dead");
            iBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
            iBookManager = null;

        }
    };
```
 另外，需要binder绑定该类
 ```
  mRemoteBookManager.asBinder().linkToDeath(mDeathRecipient, 0);
 ```
 #### Messenger
 
 Messenger翻译是信使，通过它可以在不同进程间传递Message对象，通过Messenger的构造方法,可以看出一些AIDL的痕迹，所以Messenger的底层用的也是AIDL，另外它一次只处理一个请求，因此不考虑线程同步的问题。
 
 ```
 public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
 ```
 1. 服务端进程
  
 通过Handler创建一个Messenger，可以在Handler中处理消息，另外通过Messenger返回一个当前的BInder对象。
 ```
 public class MessengerService extends Service {

    /**
     * 创建一个Messenger,需要通过handler创建
     */
    private final Messenger mMessenger = new Messenger(new MessengerHandler());


    private static class MessengerHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MessengerConstants.MESSAGE_FROM_CLIENT:
                    Log.d("MessengerService", "收到 客户端消息：" + msg.getData().getString("msg"));
                    Messenger replyTo = msg.replyTo;
                    Message message = Message.obtain(null, MessengerConstants.MESSAGE_FROM_SERVER);
                    Bundle bundle=new Bundle();
                    bundle.putString("reply","已收到，稍后回复");
                    message.setData(bundle);
                    try {
                        replyTo.send(message);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    break;
            }
        }
    }

    /**
     * 通过Messenger获取Binder
     */
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
 ```
 当然需要将服务端设置为多进程
 ```
 <service
      android:name=".messenger.MessengerService"
      android:process="com.example.lijingya.androidipc.messenger.remote"
      />
 ```
 2. 客户端
  
 首先绑定服务，然后创建一个Messenger，用于处理服务端返回的消息。
 ```
 public class MessengerActivity extends AppCompatActivity {

    private Messenger mMessenger = new Messenger(new MessengerHandler());

    private static class MessengerHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MessengerConstants.MESSAGE_FROM_SERVER:
                    Log.d("MessengerActivity", "收到服务端消息  " + msg.getData().getString("reply"));
                    break;
                default:
                    break;
            }
        }
    }

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Messenger messenger = new Messenger(service);
            Message message = Message.obtain(null, MessengerConstants.MESSAGE_FROM_CLIENT);
            Bundle bundle = new Bundle();
            bundle.putString("msg", "hello");
            message.setData(bundle);
            message.replyTo = mMessenger;
            try {
                messenger.send(message);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();
    }
}
 ```
 ##### Socket
 
 Socket是套接字，分为流式套接字（TCP）和用户数据报套接字（UDP）
 
1.TCP协议
    
  TCP协议面向连接的协议，提供稳定的双向功能，需要进过三次握手，稳定性高

2.UDP协议

UPD协议是无连接的，提供不稳定的单向通信功能，不能保证数据的稳定传输。

接下来通过Socket实现跨进程通信。

###### 创建服务端
使用IntentService替代Service处理耗时任务，在onHandleIntent方法中创建Socktet并获取输入流和输出流获取客户端和向服务端输出数据。
```
public class SocketService extends IntentService {

    private static final String[] messages = {
            "你好",
            "今年多大",
            "交个朋友"
    };

    private boolean serverDestory;

    private Random random;

    public SocketService() {
        super("SocketService");
    }


    private void dealResponse(Socket socket) throws IOException {
        InputStream inputStream = socket.getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));

        OutputStream outputStream = socket.getOutputStream();
        PrintWriter writer = new PrintWriter(new OutputStreamWriter(outputStream), true);

        while (!serverDestory) {
            String content = reader.readLine();
            Log.d("SocketService", "content " + content);
            if (!TextUtils.isEmpty(content)) {
                String replyMessage = messages[random.nextInt(3)];
                writer.println(replyMessage);
            }
        }
        reader.close();
        outputStream.close();
        socket.close();
    }

    @Override
    public void onCreate() {
        super.onCreate();
        random = new Random();
    }


    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d("SocketService", "onDestroy");
        serverDestory = true;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        ServerSocket serverSocket = null;
        try {
            serverSocket = new ServerSocket(2333);
        } catch (IOException e) {
            e.printStackTrace();
        }
        Log.d("SocketService", "serverDestory " + serverDestory);
        try {
            Socket socket = serverSocket.accept();
            dealResponse(socket);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
###### 客户端

在客户端通过线程池处理耗时的连接Socket的任务，并获取输入流获取服务端发送的信息，另外当点击发送按钮时，向服务端发送消息。
```
public class SocketActivity extends AppCompatActivity implements OnClickListener {

    private TextView tvContent;
    private Button btnSend;
    private EditText etInput;

    private static final int SOCKET_CONNET = 0;
    private static final int SOCKET_MSG = 1;
    private Socket mConnectSocket;
    private Handler sMyHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case SOCKET_CONNET:
                    btnSend.setEnabled(true);
                    break;
                case SOCKET_MSG:
                    tvContent.setText(tvContent.getText() + "\n" + "服务端：" + msg.obj);
                    break;
                default:
                    break;
            }
        }
    };

    private PrintWriter writer;
    private String msg;
    private ExecutorService executorService;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_socket);
        tvContent = findViewById(R.id.tv_content);
        btnSend = findViewById(R.id.btn_send);
        etInput = findViewById(R.id.et_input);
        btnSend.setEnabled(false);
        btnSend.setOnClickListener(this);
        Intent intent = new Intent(this, SocketService.class);
        startService(intent);
        executorService = Executors.newFixedThreadPool(10);
        executorService.execute(new ConnectTask());
    }

    class ConnectTask implements Runnable {

        @Override
        public void run() {
            connectSocket();
        }
    }

    private void connectSocket() {
        Socket socket = null;
        while (socket == null) {
            try {
                socket = new Socket("localhost", 2333);
                mConnectSocket = socket;
                writer = new PrintWriter(new BufferedWriter(new OutputStreamWriter(socket.getOutputStream())), true);
                sMyHandler.sendEmptyMessage(SOCKET_CONNET);
                Log.d("SocketActivity", "socket connect Success");
            } catch (IOException e) {
                SystemClock.sleep(2000);
                e.printStackTrace();

                Log.d("SocketActivity", "socket connect retry");
            }
        }
        try {
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            while (!SocketActivity.this.isFinishing()) {
                String content = bufferedReader.readLine();
                Log.d("SocketActivity", "content " + content);
                if (!TextUtils.isEmpty(content)) {
                    sMyHandler.obtainMessage(SOCKET_MSG, content).sendToTarget();
                }
            }

            bufferedReader.close();
            writer.close();
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onClick(View view) {
        msg = etInput.getText().toString().trim();
        if (!TextUtils.isEmpty(msg) && writer != null) {
            executorService.execute(new PrintTask());
            tvContent.setText(tvContent.getText() + "\n" + "我：" + msg);
            etInput.setText("");
        }
    }

    @Override
    protected void onDestroy() {
        if (mConnectSocket != null) {
            try {
                mConnectSocket.shutdownInput();
                mConnectSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        super.onDestroy();
    }

    class PrintTask implements Runnable {

        @Override
        public void run() {
            writer.println(msg);
        }
    }
}
```
最后使用Socket需要添加权限：
```
 <uses-permission android:name="android.permission.INTERNET"/>
  <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

##### ContentProvider

ContentProvider是安卓中的四大组件之一，因此使用的时候需要在清单文件中注册，另外ContentProvide可以用于不同应用间提供数据，底层实现也是Binder，内部除了onCreate方法，其余的增删改查的方法都运行在Binder池中，因此使用时需要处理同步问题，ContentProvider除了支持表格数据，还支持文件数据。在平常中一般都是用表格数据。在这章就通过实现一个数据库提供数据实现跨进程通信。
###### BookDbHelper.class
在该文件中创建了book_provider.db库，并创建了两张表。
```
public class BookDbHelper extends SQLiteOpenHelper {

    public static final String DB_NAME = "book_provider.db";
    public static final String BOOK_TABLE_NAME = "book";
    public static final String USER_TABLE_NAME = "user";

    private static final int DB_VERSION = 1;

    private String CREATE_BOOK_TABLE = "CREATE TABLE IF NOT EXISTS " + BOOK_TABLE_NAME + "(_id INTEGER PRIMARY KEY," + "name TEXT)";

    private String CREATE_USER_TABLE = "CREATE TABLE IF NOT EXISTS " + USER_TABLE_NAME + "(_id INTEGER PRIMARY KEY," + "name TEXT," + "sex INT)";

    public BookDbHelper(@Nullable Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL(CREATE_BOOK_TABLE);
        sqLiteDatabase.execSQL(CREATE_USER_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {

    }
}
```
###### BookProvider.class
BookProvider中制定了两张表的uri并与BookProvider的AUTHORITY 和uriCode绑定，这样就可以通过contentUri查找时知道要查询的表。
另外除了query方法像insert ，update 和delete等会影响数据改变的需要调用getContentResolver().notifyChange的方法。通知改变。
```
public class BookProvider extends ContentProvider {

    private static final String TAG = "BookProvider";

    private static final String AUTHORITY = "com.example.lijingya.androidipc.provider.book";

    public static final Uri BOOK_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/book");

    public static final Uri USER_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/user");

    private static final int BOOK_URI_CODE = 0;
    private static final int USER_URI_CODE = 1;

    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
        uriMatcher.addURI(AUTHORITY, "user", USER_URI_CODE);
    }

    private Context context;

    private SQLiteDatabase mDb;

    private String getTableName(Uri uri) {
        String tableName = null;
        switch (uriMatcher.match(uri)) {
            case BOOK_URI_CODE:
                tableName = BookDbHelper.BOOK_TABLE_NAME;
                break;
            case USER_URI_CODE:
                tableName = BookDbHelper.USER_TABLE_NAME;
                break;
        }
        return tableName;
    }

    @Override
    public boolean onCreate() {
        Log.d(TAG, "onCreate:" + Thread.currentThread().getName());
        context = getContext();
        mDb = new BookDbHelper(context).getWritableDatabase();
        ExecutorService executorService = Executors.newScheduledThreadPool(10);
        executorService.execute(new WorkRunnable());
        return true;
    }

    class WorkRunnable implements Runnable {

        @Override
        public void run() {
            initProviderData();
        }
    }

    private void initProviderData() {
        mDb.execSQL("delete from " + BookDbHelper.BOOK_TABLE_NAME);
        mDb.execSQL("delete from " + BookDbHelper.USER_TABLE_NAME);
        mDb.execSQL("insert into book values(1,'Java');");
        mDb.execSQL("insert into book values(2,'Php');");
        mDb.execSQL("insert into book values(3,'Python');");
        mDb.execSQL("insert into user values(1,'jake',1);");
        mDb.execSQL("insert into user values(1,'tom',2);");
    }


    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] strings, @Nullable String s, @Nullable String[] strings1, @Nullable String s1) {
        Log.d(TAG, "query:" + Thread.currentThread().getName());
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("不支持的Uri");
        }
        return mDb.query(tableName, strings, s, strings1, null, null, s1, null);
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues contentValues) {
        Log.d(TAG, "insert:" + Thread.currentThread().getName());
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("不支持的Uri");
        }
        mDb.insert(tableName, null, contentValues);
        context.getContentResolver().notifyChange(uri, null);
        return uri;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String s, @Nullable String[] strings) {
        Log.d(TAG, "delete:" + Thread.currentThread().getName());
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("不支持的Uri");
        }
        int count = mDb.delete(tableName, s, strings);
        if (count > 0) {
            context.getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues contentValues, @Nullable String s, @Nullable String[] strings) {
        Log.d(TAG, "update:" + Thread.currentThread().getName());
        String tableName = getTableName(uri);
        if (tableName == null) {
            throw new IllegalArgumentException("不支持的Uri");
        }
        int row = mDb.update(tableName, contentValues, s, strings);
        if (row > 0) {
            context.getContentResolver().notifyChange(uri, null);
        }
        return row;
    }

}
```
清单文件注册，并设置唯一的authorities，添加了权限，当其他引用访问该Provider时需要添加该权限另外还可以设置读写权限，并单独设置进程。
```
  <provider
      android:authorities="com.example.lijingya.androidipc.provider.book"
      android:name=".contentprovider.BookProvider"
      android:permission="com.example.lijingya.androidipc.PROVIDER"
      android:process=":provider"
      />
```
##### BookProviderActivity.class

该方法中，点击insert按钮时，插入一本书，并调用query方法查询。
```
public class BookProviderActivity extends AppCompatActivity implements OnClickListener {

    private static final String TAG = "BookProviderActivity";

    private Button btnInsert, btnDelete, btnUpdate, btnQuery;
    private Uri uri;

    @Override

    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_provider);
        uri = Uri.parse("content://com.example.lijingya.androidipc.provider.book");
        btnInsert = findViewById(R.id.btn_insert);
        btnDelete = findViewById(R.id.btn_delete);
        btnUpdate = findViewById(R.id.btn_update);
        btnQuery = findViewById(R.id.btn_query);
        btnInsert.setOnClickListener(this);
        btnDelete.setOnClickListener(this);
        btnUpdate.setOnClickListener(this);
        btnQuery.setOnClickListener(this);

    }

    @SuppressLint("Recycle")
    @Override
    public void onClick(View view) {
        switch (view.getId()) {
            case R.id.btn_insert:
                ContentValues insertValues = new ContentValues();
                insertValues.put("_id", 4);
                insertValues.put("name", "Android");
                getContentResolver().insert(BookProvider.BOOK_CONTENT_URI, insertValues);
                Cursor query = getContentResolver().query(BookProvider.BOOK_CONTENT_URI, new String[]{"_id", "name"}, null, null, null);
                while (query.moveToNext()) {
                    Book book = new Book();
                    book.setId(query.getInt(0));
                    book.setName(query.getString(1));
                    Log.d(TAG, "book:" + book.toString());
                }
                query.close();
                break;
            case R.id.btn_delete:
                break;
            case R.id.btn_update:
                break;
            case R.id.btn_query:
//                getContentResolver().query(uri, null, null, null, null);
//                getContentResolver().query(uri, null, null, null, null);
//                getContentResolver().query(uri, null, null, null, null);
                break;
        }
    }
}
```
