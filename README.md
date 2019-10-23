正文大纲
一、 概念QA以及前置技能
二、传统方式IPC通信写法 与 使用IPC框架进行RPC通信 的对比
三、Demo展示
四、框架核心思想讲解
五、 写在最后的话

正文
一、 概念QA以及前置技能
Q:什么时候会用到多进程通信？
A: 常见的多进程app一般是大型公司的 app组，像是腾讯系的 QQ 微信 QQ空间，QQ邮箱等等，有可能 在QQ邮箱登录时，可以直接调用QQ的登录服务，另外，腾讯阿里都有小程序，作为一个第三方开发的小程序应用，在微信客户端运行，如果和微信放在同一个进程运行，一旦崩溃，微信也跟着玩完，明明是小程序开发者的锅，硬是让腾讯给背了，不合适。
而小型公司，emmmmm,连多进程开发都用的很少，就不要说通信了。但是，如果没有一颗进大厂的心，就学不到高阶技能，有些东西学了，总比一无所知要好。

Q:使用多进程有什么好处？
A:
1）进程隔离，子app崩溃，不会影响其他进程。
2）系统运行期间，对每个进程的内存划分是有一个上限的，具体多少，视具体设备而定，利用多进程开发，可以提高程序的可运行内存限制。
3）如果系统运行期间内存吃紧，可以杀死子进程，减少系统压力。杀死进程的方式，往往比优化单个app的内存更加直接有效

Q:什么叫RPC？
A：从客户端上通过参数传递的方式调用服务器上的一个函数并得到返回的结果，隐藏底层的通讯细节。在使用形式上像调用本地函数一样去调用远程函数。

Q:我们自己定义一个RPC进程间通信框架，有什么实际用处？
A:定义框架的作用，都是 把脏活，累活，别人不愿意重复干的活，都放到框架里面去，让使用者用最干净的方式使用业务接口。定义一个RPC进程间通信框架，可以把C/S两端那些恶心人的AIDL编码都集中放到框架module中，这是最直观的好处，另外，客户端原本还需要手动去bindService，定义ServiceConnection，取得Binder，再去通信，使用RPC框架，这些内容都可以放到框架module中. 而C/S两端的代码，就只剩下了 S端的服务注册，C端的RPC接口调用，代码外观上非常简洁（可能这里文字描述不够直观，后面有图）

前置技能
要理解本文的核心代码，还是需要一些基础的，大致如下：
四大组件之一Service使用方法，
android AIDL通信机制,
java注解，java反射，java 泛型

二、传统方式IPC通信写法 与 使用IPC框架进行RPC通信 的对比
见github : https://github.com/18598925736/MyRpcFramework , 运行 aidl_client和 aidl_service

先展示效果

aidl.gif
![](https://github.com/wangkangda11/MyRpcFramework-master\app\src\main\res\mipmap-hdpi\ic_launcher.png)  


图中的 查找用户，是从 服务端读取的数据，观察一下核心代码：
这是我优化之后的IPC项目结构(如果不优化，那么客户端 服务端都需要编写一样的AIDL代码，还要有一个包括包名在内神马都要一模一样的JavaBean，实在是丑陋):

image.png
服务端核心代码:

public class ServiceActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        startService(new Intent(this, MyService.class));//服务端，app启动之后，自动启动服务
    }
}
public class MyService extends Service {

    ConcurrentMap<String, UserInfoBean> map;

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        map = new ConcurrentHashMap<>();
        for (int i = 0; i < 100; i++) {
            map.put("name" + i, new UserInfoBean("name" + i, "accountNo" + i, i));
        }
        return new IUserInfo.Stub() {//数据接收器 Stub
            @Override
            public UserInfoBean getInfo(String name) {
                return map.get(name);
            }
        };
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.e("MyService", "onCreate: success");
    }
}
客户端核心代码 ：

public class ClientActivity extends AppCompatActivity {

    private TextView resultView;
    private String TAG = "clientLog";

    private int i = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initView();
    }

    private void initView() {
        resultView = findViewById(R.id.resultView);
        findViewById(R.id.connect).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                bindService();
            }
        });
        findViewById(R.id.disconnect).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    unbindService(connection);
                    resultView.setText("尝试释放");
                } catch (IllegalArgumentException e) {
                    resultView.setText("已经释放了");
                }
            }
        });
        findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (iUserInfo != null) {
                    try {
                        ((Button) v).setText("查找name为:name" + ((i++) + 1) + "的UserInfoBean");
                        UserInfoBean bean = iUserInfo.getInfo("name" + i);
                        if (bean != null)
                            resultView.setText(bean.toString());
                        else
                            resultView.setText("没找到呀");
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                } else {
                    resultView.setText("没有连接上service");
                }
            }
        });
    }

    //作为IPC的客户端，我们需要 建立起和Service的连接
    private IUserInfo iUserInfo;//操作句柄，可以通过它向service发送数据

    private void bindService() {
        if (iUserInfo == null) {
            Intent intent = new Intent();
            intent.setComponent(new ComponentName(
                    "study.hank.com.aidl_service",
                    "study.hank.com.aidl_service.MyService"));
            bindService(intent, connection, Context.BIND_AUTO_CREATE);
            resultView.setText("尝试连接");
        } else {
            resultView.setText("已经连接上service" + iUserInfo);
        }
    }

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            iUserInfo = IUserInfo.Stub.asInterface(service);
            resultView.setText("连接成功");
            Log.d(TAG, "connection:" + "连接成功");
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            iUserInfo = null;
            resultView.setText("连接 已经断开");
            Log.d(TAG, "connection:" + "已经断开");
        }
    };

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(connection);
    }
}
很容易发现，服务端的代码量尚可，不是很复杂，但是客户端这边，要处理 connection，要手动去绑定以及解绑Service,所有参与通信的javabean还必须实现序列化接口parcelable
Demo中只有一个客户端，还不是很明显，但是如果有N个客户端Activity都需要与service发生通信，意味着每一个Activity都必须写类似的代码. 不但累赘，而且丑陋.

前方高能
不使用RPC框架时，CS两端的代码的结构，已经有了大致的印象，下面是 使用IPC框架时 客户端、服务端 的核心代码

客户端

客户端核心代码.png

之前的bindService呢？没了。客户端使用此框架来进行 进程通信，不用去关心AIDL怎么写了，不用关注bindService，ServiceConnection，省了很多事。
服务端

服务端核心代码.png
对比 使用框架前后，我们的核心代码的变化
有什么变化？显而易见，极大缩减了客户端的编码量，而且，一劳永逸，除非需求大改，不然这个框架，一次编写，终身使用。除此之外，使用框架还可以极大地节省客户端代码量，减少人为编码时产生的可能疏漏（比如忘记释放连接造成泄漏等）. 试想一下，如果你是一个团队leader，团队成员的水平很有可能参差不齐，那么如何保证项目开发中 出错概率最小 ------- 使用框架, 用框架来简化团队成员的编码量和编码难度，让他们傻瓜式地写代码.

三、Demo展示
github地址：https://github.com/18598925736/MyRpc


ipc.gif

以上Demo，模拟的场景是：
服务端：开启一个登录服务，启动服务之后，保存一个可以登录的用户名和密码
客户端1：RPC调用登录服务，用户名和密码 和服务端的一样，可以登录成功
客户端2：RPC调用登录服务，用户名和密码 和服务端的不一样，登录失败
Demo工程代码结构图

客户端.png


服务端.png


框架层.png
注：客户端和服务端必须同时依赖框架层module implementation project(":ipc")

四、框架核心思想讲解
我们不使用IPC框架时，有两件事非常恶心：

1. 随着业务的扩展，我们需要频繁(因为要新增业务接口)改动AIDL文件，而且AIDL修改起来没有任何代码提示,只有到了编译之后，编译器才会告诉我哪里错了，而且 直接引用到的JavaBean还必须手动再声明一次。实在是不想在这个上面浪费时间。
2. 所有客户端Activity，只要想进行进程间binder通信，就不可避免要去手动bindService,随后去处理 Binder连接，重写ServiceConnection，还要在适当的时候释放连接，这种业务不相关而且重复性很大的代码，要尽量少写。

IPC框架将会着重解决这两个问题。下面开始讲解核心设计思想

注：
1.搭建框架牵涉的知识面会很广，我不能每个细节都讲得很细致，一些基础部分一笔带过的，如有疑问，希望能留言讨论。
2.设计思路都是环环相扣的，阅读时最好是从上往下依次理解.

框架思想四部曲：
1）业务注册
上文说到，直接使用AIDL通信，当业务扩展时，我们需要对AIDL文件进行改动，而改起来比较费劲，且容易出错。怎么办？利用 业务注册的方式,将 业务类的class对象，保存到服务端 内存中。
进入Demo代码 Registry.java：

public class Ipc {

    /**
     * @param business
     */
    public static void register(Class<?> business) {
        //注册是一个单独过程，所以单独提取出来，放在一个类里面去
        Registry.getInstance().register(business);//注册机是一个单例，启动服务端，
        // 就会存在一个注册机对象，唯一，不会随着服务的绑定解绑而受影响
    }
  ...省略无关代码
}
/**
 * 业务注册机
 */
public class Registry {
    ...省略不关键代码

    /**
     * 业务表
     */
    private ConcurrentHashMap<String, Class<?>> mBusinessMap
            = new ConcurrentHashMap<>();
    /**
     * 业务方法表, 二维map，key为serviceId字符串值，value为 一个方法map  - key，方法名；value
     */
    private ConcurrentHashMap<String, ConcurrentHashMap<String, Method>> mMethodMap
            = new ConcurrentHashMap<>();

    /**
     * 业务类的实例,要反射执行方法，如果不是静态方法的话，还是需要一个实例的，所以在这里把实例也保存起来
     */
    private ConcurrentHashMap<String, Object> mObjectMap = new ConcurrentHashMap<>();

    /**
     * 业务注册
     * 将业务class的class和method对象都保存起来，以便后面反射执行需要的method
     */
    public void register(Class<?> business) {
        //这里有个设计，使用注解，标记所使用的业务类是属于哪一个业务ID，在本类中，ID唯一
        ServiceId serviceId = business.getAnnotation(ServiceId.class);//获取那个类头上的注解
        if (serviceId == null) {
            throw new RuntimeException("业务类必须使用ServiceId注解");
        }
        String value = serviceId.value();
        mBusinessMap.put(value, business);//把业务类的class对象用 value作为key，保存到map中

        //然后要保存这个business类的所有method对象
        ConcurrentHashMap<String, Method> tempMethodMap = mMethodMap.get(value);//先看看方法表中是否已经存在整个业务对应的方法表
        if (tempMethodMap == null) {
            tempMethodMap = new ConcurrentHashMap<>();//不存在，则new
            mMethodMap.put(value, tempMethodMap);// 并且将它存进去
        }
        for (Method method : business.getMethods()) {
            String methodName = method.getName();
            Class<?>[] parameterTypes = method.getParameterTypes();
            String methodMapKey = getMethodMapKeyWithClzArr(methodName, parameterTypes);
            tempMethodMap.put(methodMapKey, method);
        }
         ...省略不关键代码
    }

   ...省略不关键代码

    /**
     * 如何寻找到一个Method？
     * 参照上面的构建过程，
     *
     * @param serviceId
     * @param methodName
     * @param paras
     * @return
     */
    public Method findMethod(String serviceId, String methodName, Object[] paras) {
        ConcurrentHashMap<String, Method> map = mMethodMap.get(serviceId);
        String methodMapKey = getMethodMapKeyWithObjArr(methodName, paras); //同样的方式，构建一个StringBuilder
        return map.get(methodMapKey);
    }

    /**
     * 放入一个实例
     *
     * @param serviceId
     * @param object
     */
    public void putObject(String serviceId, Object object) {
        mObjectMap.put(serviceId, object);
    }

    /**
     * 取出一个实例
     *
     * @param serviceId
     */
    public Object getObject(String serviceId) {
        return mObjectMap.get(serviceId);
    }
}
/**
 * 自定义注解，用于注册业务类的
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ServiceId {
    String value();
}
我利用一个单例的Registry类，将当前这个业务class对象，拆解出每一个Method，保存到map集合中。
而，保存这些Class，Method，则是为了 反射执行指定的业务Method做准备。
此处有几个精妙设计：
1， 利用自定义注解 @ServiceId 对业务接口和实现类，都形成约束，这样业务实现类就有了进行唯一性约束，因为在Registry类中，一个ServiceId只针对一种业务,如果用Registry类注册一个没有@ServiceId注解的业务类，就会抛出异常。


image.png
2， 利用注解@ServiceId的value作为key，保存所有的业务实现类的Class ， 以及该Class的所有public的Method 到map集合中，通过日志打印，很容易看出当前服务端有哪些 业务类，业务类有哪些可供外界调用的方法。(·这里需要注意，保存方法时，必须连同方法的参数类型一起作为key，因为存在同名方法重载的情况·)
当你运行Demo，启动服务端的时候，过滤一下日志,就能看到：


image.png

3 ，如果再发生 业务扩展的情况，我们只需要直接改动加了@ServiceId注解的业务类即可，并没有其他多余的动作。
如果我在IUserBusiness接口中，增加一个logout方法，并且在实现类中去实现它。那么，再次启动服务端app，上图的日志中就会多出一个logout方法.

image.png

4，提供一个Map集合，专门用来保存每一个ServiceId对应的Object，并提供getObject和putObject方法，以便反射执行Method时所需。

image.png

OK,一切准备万全。业务类的每个部分基本上都保存到了服务端进程的内存中，反射执行Method，随时可以取用。
2）自定义通信协议
跨进程通信，我们本质上还是使用Binder AIDL这一套，所以AIDL代码还是要写的，但是，是写在框架层中，一旦确定了通信协议，那这一套AIDL就不会随着业务的变动去改动它，因为它是框架层代码，不会随意去动。 要定自己的通信协议，其实没那么复杂。想一想，通信，无非就是客户端向服务端发送消息，并且取得回应的过程，那么，核心方法就确定为 send：


image.png

入参是Request，返回值是Response，有没有觉得很像HTTP协议。
request和response都是我们自定义的，注意，要参与跨进程通信的javaBean，必须实现Parcelable接口，它们的属性类型也必须实现Parcelable接口。

image.png
Request中的重要元素包括：
serviceId 客户端告诉服务端要调用哪一个业务
methodName 要调用哪一个方法
parameters 调这个方法要传什么参数
这3个元素，足矣涵盖客户端的任何行为。但是，由于我的业务实现类定义 为了单例，所以它有一个静态的getInstance方法。静态方法和普通方法的反射调用不太一样，所以，加上一个type属性，加以区分。

public class Request implements Parcelable {
    private int type;
    /**
     * 创建业务类实例,并且保存到注册表中
     */
    public final static int TYPE_CREATE_INSTANCE = 0;
    /**
     * 执行普通业务方法
     */
    public final static int TYPE_BUSINESS_METHOD = 1;

    public int getType() {
        return type;
    }
    private String serviceId;  //客户端告诉服务端要调用哪一个业务
    private String methodName;//要调用哪一个方法
    private Parameter[] parameters;//调这个方法要传什么参数
    ...省略无关代码
}
Response中的重要元素有：
result 字符串类型，用json字符串表示接口执行的结果
isSuccess 为 true，接口执行成功，false 执行失败

public class Response implements Parcelable {
    private String result;//结果json串
    private boolean isSuccess;//是否成功
}
最后，Request引用的Parameter类：
type 表示，参数类型（如果是String类型，那么这个值就是 java.long.String）
value 表示，参数值，Gson序列化之后得到的字符串

public class Parameter implements Parcelable {
    private String value;//参数值序列化之后的json
    private String type;//参数类型 obj.getClass
}
为什么设计这么一个Parameter？为什么不直接使用Object？
因为，Request 中需要 客户端给的参数列表，可是如果直接使用客户端给的Object[] ，你并不能保证数组中的所有参数都实现了Parcelable，一旦有没有实现的，通信就会失败（binder AIDL通信，所有参与通信的对象，都必须实现Parcelable，这是基础）,所以，直接用gson将Object[] 转化成Parameter[],再传给Request，是不错的选择，当需要反射执行的时候，再把Parameter[] 反序列化成为 Object[] 即可。

OK,通信协议的3个类讲解完了，那么下一步应该是把这个协议使用起来。

3）binder连接封装
参照Demo源码，这一个步骤中的两个核心类：IpcService , Channel

先说 IpcService.java
它就是一个 extends android.app.Service 的一个普通Service，它在服务端启动，然后与客户端发生通信。它必须在服务端app的 manifest文件中注册。同时，当客户端与它连接成功时，它必须返回一个Binder对象，所以我们要做两件事：
1 服务端的manifest中对它进行注册

image.png

ps: 这里肯定有人注意到了，上面service注册时，其实使用了多个IpcService的内部静态子类，设计多个内部子类的意义是，考虑到服务端存在多个 业务接口的存在，让每一个业务接口的实现类 都由一个专门的IpcService服务区负责通信。
举个例子：上图中存在两个 IpcService的子类，我让IpcService0 负责 用户业务UserBusiness，让IpcService1 负责 DownloadBusiness, 当 客户端需要使用UserBusiness时，就连接到IpcService0，当需要使用 DownloadBusiness时，就连接到IpcService1.
但是这个并不是硬性规定，而只是良好的编程习惯，一个业务接口A，对应一个IpcService子类A，客户端要访问业务接口A，就直接和IpcService子类A通信即可。
同理，一个业务接口B，对应一个IpcService子类B，客户端要访问业务接口B，就直接和IpcService子类B通信即可。(我是这么理解的，如有异议，欢迎留言)
2 重写onBind方法，返回一个Binder对象：
我们要明确返回的这个Binder对象的作用是什么。
它是给客户端去使用的，客户端用它来调用远程方法用的，所以，我们前面两个大步骤准备的 注册机Registry，和通信协议 request,response，就是在这里大显身手了 .

public IBinder onBind(Intent intent) {
        return new IIpcService.Stub() {//返回一个binder对象,让客户端可以binder对象来调用服务端的方法
            @Override
            public Response send(Request request) throws RemoteException {
                //当客户端调用了send之后
                //IPC框架层应该要 反射执行服务端业务类的指定方法,并且视情况返回不同的回应
                //客户端会告诉框架，我要执行哪个类的哪个方法，我传什么参数
                String serviceId = request.getServiceId();
                String methodName = request.getMethodName();
                Object[] paramObjs = restoreParams(request.getParameters());
                //所有准备就绪，可以开始反射调用了？
                //先获取Method
                Method method = Registry.getInstance().findMethod(serviceId, methodName, paramObjs);
                switch (request.getType()) {
                    case Request.TYPE_CREATE_INSTANCE:
                        try {
                            Object instance = method.invoke(null, paramObjs);
                            Registry.getInstance().putObject(serviceId, instance);
                            return new Response("业务类对象生成成功", true);
                        } catch (Exception e) {
                            e.printStackTrace();
                            return new Response("业务类对象生成失败", false);
                        }
                    case Request.TYPE_BUSINESS_METHOD:
                        Object o = Registry.getInstance().getObject(serviceId);
                        if (o != null) {
                            try {
                                Log.d(TAG, "1:methodName:" + method.getName());
                                for (int i = 0; i < paramObjs.length; i++) {
                                    Log.d(TAG, "1:paramObjs     " + paramObjs[i]);
                                }
                                Object res = method.invoke(o, paramObjs);
                                Log.d(TAG, "2");
                                return new Response(gson.toJson(res), true);
                            } catch (Exception e) {
                                return new Response("业务方法执行失败" + e.getMessage(), false);
                            }
                        }
                        Log.d(TAG, "3");
                        break;
                }
                return null;
            }
        };
    }
这里有一些细节需要总结一下：
1 从request中拿到的 参数列表是Parameter[]类型的，而我们反射执行某个方法，要的是Object[]，那怎么办？反序列化咯，先前是用gson去序列化的，这里同样使用gson去反序列化, 我定义了一个名为：restoreParams的方法去反序列化成Object[].
2 之前在request中，定义了一个type，用来区分静态的getInstance方法，和 普通的业务method，这里要根据request中的type值，区分对待。getInstance方法，会得到一个业务实现类的Object，我们利用Registry的putObject把它保存起来。 而，普通method，再从Registry中将刚才业务实现类的Object取出来，反射执行method
3 静态getInstance的执行结果，不需要告知客户端，所以没有返回Response对象，而 普通Method，则有可能存在返回值，所以必须将返回值gson序列化之后，封装到Response中，return出去。

再来讲 Channel类：

之前抱怨过，不喜欢重复写 bindService，ServiceConnection，unbindService。但是其实还是要写的，写在IPC框架层，只写一次就够了。

public class Channel {
    String TAG = "ChannelTag";
    private static final Channel ourInstance = new Channel();

    /**
     * 考虑到多重连接的情况，把获取到的binder对象保存到map中，每一个服务一个binder
     */
    private ConcurrentHashMap<Class<? extends IpcService>, IIpcService> binders = new ConcurrentHashMap<>();

    public static Channel getInstance() {
        return ourInstance;
    }

    private Channel() {
    }

    /**
     * 考虑app内外的调用,因为外部的调用需要传入包名
     */
    public void bind(Context context, String packageName, Class<? extends IpcService> service) {
        Intent intent;
        if (!TextUtils.isEmpty(packageName)) {
            intent = new Intent();
            Log.d(TAG, "bind:" + packageName + "-" + service.getName());
            intent.setClassName(packageName, service.getName());
        } else {
            intent = new Intent(context, service);
        }
        Log.d(TAG, "bind:" + service);
        context.bindService(intent, new IpcConnection(service), Context.BIND_AUTO_CREATE);
    }

    private class IpcConnection implements ServiceConnection {

        private final Class<? extends IpcService> mService;


        public IpcConnection(Class<? extends IpcService> service) {
            this.mService = service;
        }

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IIpcService binder = IIpcService.Stub.asInterface(service);
            binders.put(mService, binder);//给不同的客户端进程预留不同的binder对象
            Log.d(TAG, "onServiceConnected:" + mService + ";bindersSize=" + binders.size());
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            binders.remove(mService);
            Log.d(TAG, "onServiceDisconnected:" + mService + ";bindersSize=" + binders.size());
        }

    }

    public Response send(int type, Class<? extends IpcService> service, String serviceId, String methodName, Object[] params) {
        Response response;
        Request request = new Request(type, serviceId, methodName, makeParams(params));
        Log.d(TAG, ";bindersSize=" + binders.size());
        IIpcService iIpcService = binders.get(service);
        try {
            response = iIpcService.send(request);
            Log.d(TAG, "1 " + response.isSuccess() + "-" + response.getResult());
        } catch (RemoteException e) {
            e.printStackTrace();
            response = new Response(null, false);
            Log.d(TAG, "2");
        } catch (NullPointerException e) {
            response = new Response("没有找到binder", false);
            Log.d(TAG, "3");
        }
        return response;
    }
    ...省略不关键代码
}
上面的代码是Channel类代码，两个关键：
1 bindService+ServiceConnection 供客户端调用，绑定服务，并且将连接成功之后的binder保存起来


image.png

2 提供一个send方法，传入request，且 返回response，使用serviceId对应的binder 完成通信。
4）动态代理实现RPC
终于到了最后一步，前面3个步骤，为进程间通信做好了所有的准备工作，只差最后一步了------ 客户端调用服务。
重申一下RPC的定义：让客户端像 使用本地方法一样 调用远程过程。

像 使用本地方法一样？我们平时是怎么使用本地方法的呢？

A a = new A();
a.xxx();
类似上面这样。
但是我们的客户端和服务端是两个隔离的进程,内存并不能共享，也就是说 服务端存在的 类对象，不能直接被客户端使用，那怎么办？泛型+动态代理！
我们需要构建一个处在客户端进程内的 业务代理类对象,它可以执行和 服务端的 业务类 一样的方法，但是它确实不是 服务端进程的那个对象，如何实现这种效果？

public class Ipc {

    ...省略无关代码

    /**
     * @param service
     * @param classType
     * @param getInstanceMethodName
     * @param params
     * @param <T>                   泛型，
     * @return
     */
    public static <T> T getInstanceWithName(Class<? extends IpcService> service,
                                            Class<T> classType, String getInstanceMethodName, Object... params) {

        //这里之前不是创建了一个binder么，用binder去调用远程方法，在服务端创建业务类对象并保存起来
        if (!classType.isInterface()) {
            throw new RuntimeException("getInstanceWithName方法 此处必须传接口的class");
        }
        ServiceId serviceId = classType.getAnnotation(ServiceId.class);
        if (serviceId == null) {
            throw new RuntimeException("接口没有使用指定ServiceId注解");
        }
        Response response = Channel.getInstance().send(Request.TYPE_CREATE_INSTANCE, service, serviceId.value(), getInstanceMethodName, params);
        if (response.isSuccess()) {
            //如果服务端的业务类对象创建成功，那么我们就构建一个代理对象，实现RPC
            return (T) Proxy.newProxyInstance(
                    classType.getClassLoader(), new Class[]{classType},
                    new IpcInvocationHandler(service, serviceId.value()));
        }
        return null;
    }
}
上面的getInstanceWithName，会返回一个动态代理的 业务类对象(处在客户端进程), 它的行为 和 真正的业务类(服务端进程)一模一样。
这个方法有4个参数
@param service 要访问哪一个远程service，因为不同的service会返回不同的Binder
@param classType 要访问哪一个业务类，注意，这里的业务类完全是客户端自己定义的，包名不必和服务端一样，但是一定要有一个和服务端对应类一样的注解。注解相同，框架就会认为你在访问相同的业务。
@param getInstanceMethodName 我们的业务类都是设计成单例的，但是并不是所有获取单例对象的方法都叫做getInstance，我们框架要允许其他的方法名
@param params 参数列表，类型为Object[]

重中之重，实现RPC的最后一个步骤，如图：


image.png

如果服务端的单例对象创建成功，那么说明 服务端的注册表中已经存在了一个业务实现类的对象，进而，我可以通过 binder通信来 使用这个对象 执行我要的业务方法，并且拿到方法返回值，最后 把返回值反序列化成为Object ，作为动态代理业务类的方法的执行结果。
关键代码 IpcInvocationHandler :

/**
 * RPC调用 执行远程过程的回调
 */
public class IpcInvocationHandler implements InvocationHandler {

    private Class<? extends IpcService> service;
    private String serviceId;
    private static Gson gson = new Gson();


    IpcInvocationHandler(Class<? extends IpcService> service, String serviceId) {
        this.service = service;
        this.serviceId = serviceId;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        //当，调用代理接口的方法时，就会执行到这里，执行真正的过程
        //而你真正的过程是远程通信
        Log.d("IpcInvocationHandler", "类：" + serviceId + "          方法名" + method.getName());
        for (int i = 0; i < args.length; i++) {
            Log.d("IpcInvocationHandler", "参数：" + args.getClass().getName() + "/" + args[i].toString());
        }
        Response response = Channel.getInstance().send(Request.TYPE_BUSINESS_METHOD, service, serviceId, method.getName(), args);
        if (response.isSuccess()) {
            //如果此时执行的方法有返回值
            Class<?> returnType = method.getReturnType();
            if (returnType != void.class && returnType != Void.class) {
                //既然有返回值，那就必须将序列化的返回值 反序列化成对象
                String resStr = response.getResult();
                return gson.fromJson(resStr, returnType);
            }
        }

        return null;
    }
}
ok，收工之前总结一下，最后RPC的实现，借助了Proxy动态代理+Binder通信。 用动态代理产生一个本进程中的对象，然后在重写invoke时，使用binder通信执行服务端过程拿到返回值。这个设计确实精妙。

五、 写在最后的话
本案例提供的两个Demo，都只是作为演示效果作用的，代码不够精致，请各位不要在意这些细节.
此框架并非本人原创，课题内容来自 享学课堂Lance老师,本文只做学习交流之用，转载请务必注明出处，谢谢合作。
第二个Demo（IPC通信框架实现RPC），我的原装代码中只实现了服务端 1个服务，2个客户端同时调用，但是这个框架是支持服务端多个服务，多个客户端同时调用的，所以，可以尝试在我的代码基础上扩展出服务端N个业务接口和实现类，多个客户端混合调用的场景。应该不会有bug。
作者：波澜步惊
链接：https://www.jianshu.com/p/0c84d7d44f80
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
