Android 进程间通信-Intent、Messenger、AIDL
===

​Android进程间通信（IPC，Inter-Process Communication）底层采用的是 Binder 机制，具体到应用层有网友根据安卓四大组件将进程间通信方式分为对应的四种方式 Activity, Broadcast, ContentProvider, Service。Activity可以跨进程调用其他应用程序的Activity；Content Provider可以跨进程访问其他应用程序中的数据（以Cursor对象形式返回）；Broadcast可以向android系统中所有应用程序发送广播，而需要跨进程通讯的应用程序可以监听这些广播；Service可通过 AIDL(Android Interface Definition Language) 实现跨进程通信。

本文根据实现 IPC 的具体手段，分为以下四种方式：

* 使用 Intent 通信
* 持久化数据通信（ContentProvider，本地文件等）
* 使用信使（Messenger）通信
* 使用 AIDL 通信

使用 Intent 通信
---

Intent 是安卓的一种消息传递机制，可用于 Activity/Service 交互，传送广播（Broadcast）事件到 BroadcastReceiver。当交互的 Activity/Service 在不同进程内，此时 Intent 的传递便是跨进程的，。具体使用方法有：

* 通过Context.startActivity() or Activity.startActivityForResult() 启动一个Activity；
* 通过 Context.startService() 启动一个服务，或者通过Context.bindService() 和后台服务交互；
* 通过广播方法(比如 Context.sendBroadcast(),Context.sendOrderedBroadcast(),  Context.sendStickyBroadcast()) 发给broadcast receivers。

关于 Intent 的详细内容可参考文章 [Android Activity和Intent机制学习笔记](http://www.cnblogs.com/feisky/archive/2010/01/16/1649081.html)。

通过 Intent.putExtra() 可将基本类型数据， Bundle及可序列化的对象存入 Intent 中在进程间传递。其中基本类型数据包括 double，float，byte，short，int，long，CharSequence（String, CharBuffer等的父类），char，boolean，及其对应的数组。放入 extra 的对象需要实现序列化（Bundle也实现了Parcelable）。对象的序列化有两各方式：

* 实现Java的序列化接口Serializable 
* 实现Android特有的序列化接口Parcelable

Serializable 使用简单只需要注明实现即可不需要实现额外的方法，可保存对象的属性到本地文件、数据库、网络流以方便数据传输。Parcelable 相比 Serializable 效率更高更适合移动端序列化数据传递，需要手动将类的变量打包并实现必要的方法，且实现 Parcelable 的类的变量如果是对象亦需要实现了序列化。Parcelable 序列化是存储在内存的，不能适用用保存对象到本地、网络流、数据库。具体使用方法可参考文章[android Activity之间数据传递 Parcelable和Serializable接口的使用](http://blog.csdn.net/js931178805/article/details/8268144) 及 Android文档 [Parcelable](http://developer.android.com/reference/android/os/Parcelable.html)。

持久化数据通信
---
一种间接实现安卓进程间通信的方法是持久化数据，如使用数据库，ContentProvider，本地文件，SharePreference等，将数据存储在两个程序皆可获取的地方，可以间接达到程序间通信，并不能算是真正的IPC。这种方法一般效率较低，缺乏主动通知能力，一般用于程序间的简单数据交互。

使用信使（Messenger）通信
---
Messenger 在进程间通信的方式和 Hanlder-Message 类似，Hanlder在A进程中，B进程持有A的 Messenger 通过此发送 Message 到A实现进程间通信。Messenger 是对 Binder 的简单包装。相对于 AIDL 方式，Messenger 是将请求放入消息队列中然后逐条取出处理的，而纯 AIDL 接口可以将多条请求并发传递到服务端（多线程处理请求）。如果不需要并发处理请求时，采用 Messenger 是更好的选择，这种方法更为简单易于实现，不需要额外写 AIDL，不需要考虑多线程，对于 Handler-Message 机制更为广大安卓开发者熟悉不易出错。

使用 Messenger 一般步骤如下：

* 服务端的 Service 实现一个 Handler 来处理请求。
* 用该 Handler 创建一个 Messenger。
* 在服务端返回客户端的方法 onBind() 中返回 Messenger.getBinder() 创建的 IBinder。
* 客户端通过得到的 IBinder 初始化一个 Messenger（引用了服务端的 Handler），此时客户端可通过该 Messenger 发送消息到服务端。

从上面的步骤我们知道如何将服务端的信鸽传送到客户端使客户端可以发送请求到服务端，那么服务端如何将请求送达客户端呢？当客户端获得服务端的 Messenger 后，可以很轻松的将自己的 Messenger 通过 Message 发送到服务端，使服务端可以发送请求到客户端。赋值 Message.replyTo 为客户端自己的 Messenger ，再将此 Message 发送到服务端即可。

下面是使用简单的 Demo 演示了使用 Messenger 如何跨进程通信，服务端为一个 Service，客户端为Activity。通过指定 ServerService 的 `android:process` 属性使其在另一个进程。
服务端进程代码如下：
	
```
public class ServerService extends Service {

    static final String TAG = "halflike";

    public static final int MSG_BIND_MESSENGER = 0;
    public static final int MSG_SAY_HELLO = 1;
    public static final int MSG_ARE_YOU_OK = 2;

    Messenger sMessenger = null;
    Messenger cMessenger = null;

    @Override
    public void onCreate() {
        super.onCreate();
        // 创建服务端信使
        sMessenger = new Messenger(new ServerHandler(this.getMainLooper()));
        Log.d(TAG, "server stated.");
    }

    @Override
    public IBinder onBind(Intent intent) {
        return sMessenger.getBinder();
    }

    class ServerHandler extends Handler {

        public ServerHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_BIND_MESSENGER:
                	 // 获得客户端信使
                    cMessenger = msg.replyTo;
                    if (cMessenger != null) {
                        Bundle data = new Bundle();
                        data.putString("content", "server:connet succeed!");
                        Message msg2c = Message.obtain(null, MSG_BIND_MESSENGER);
                        msg2c.setData(data);
                        try {
                            cMessenger.send(msg2c);
                        } catch (RemoteException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
                case MSG_SAY_HELLO:
                    if (cMessenger != null) {
                        Bundle data = new Bundle();
                        data.putString("content", "server:Hello, I am Server!");
                        Message msg2c = Message.obtain(null, MSG_SAY_HELLO);
                        msg2c.setData(data);
                        try {
                            cMessenger.send(msg2c);
                        } catch (RemoteException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }

        }
    }

}
```

客户端通过 bindService() 连接服务端并获得 Messenger，代码如下：

```
public class ClientActivity extends Activity {

    static final String TAG = "halflike";

    TextView recordView;
    Button sayHelloBtn;

    Messenger sMessenger = null;
    Messenger cMessenger = null;

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "connect server succeed.");
            // 获取服务端的 Messenger
            sMessenger = new Messenger(service);
            Message msg = Message.obtain(null, ServerService.MSG_BIND_MESSENGER);
            // 传递客户端的 Messenger 到服务端
            msg.replyTo = cMessenger;
            try {
                sMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "disconnect server");
            sMessenger = null;
        }
    };

    class ClientHandler extends Handler {
        public ClientHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case ServerService.MSG_SAY_HELLO:
                    recordView.setText(recordView.getText() + "\n" +
                            msg.getData().getString("content"));
                    break;
                case ServerService.MSG_BIND_MESSENGER:
                    recordView.setText(recordView.getText() + "\n" +
                            msg.getData().getString("content"));
                default:
                    super.handleMessage(msg);
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        recordView = (TextView) findViewById(R.id.record);
        sayHelloBtn = (Button) findViewById(R.id.sayHello);
        sayHelloBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "client:hello, server");
                if (sMessenger != null) {
                    Message msg = Message.obtain(null, ServerService.MSG_SAY_HELLO);
                    try {
                        sMessenger.send(msg);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    recordView.setText(recordView.getText() + "\n" + "client:Hi, I am client.");
                }
            }
        });
        cMessenger = new Messenger(new ClientHandler(this.getMainLooper()));
        // 启动服务端
        Intent intent = new Intent(this, ServerService.class);
        Log.d(TAG, "bind server.");
        bindService(intent, serviceConnection, BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(serviceConnection);
        super.onDestroy();
    }
}
```

使用 AIDL 通信
---



参考 
---
[Android Activity和Intent机制学习笔记](http://www.cnblogs.com/feisky/archive/2010/01/16/1649081.html)

[Intent 谷歌文档](http://developer.android.com/reference/android/content/Intent.html)

[Using a Messenger 谷歌文档](http://developer.android.com/guide/components/bound-services.html#Messenger)
