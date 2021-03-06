Android插件化、动态加载及动态更新
===
最近琢磨了 Android 插件化方面的实现，子曾曰过：好记性不如烂笔头，于是对自己近日所得做个总结。

## 基本概念
Android 插件化一般指已安装的 App 直接调用未安装的 App 或运行其类方法。
Android 插件化有以下几点好处：
1. 模块解耦
1. 突破单个 dex(Dalvik Executable) 文件不能超过65535个方法数的限制
1. 动态更新

对于第2点其实使用 Google 提供的 [multidex support library](https://developer.android.com/tools/support-library/features.html#multidex) 可轻松实现多 dex 文件拆分，关于 dex 方法数的限制可参考 Google 官方文档 [Building Apps with Over 65K Methods](https://developer.android.com/tools/building/multidex.html)。

## 类的加载
插件化中除了一般类的加载还包括 Activity, Service 等组件的加载和运行，这里只讨论单纯的类加载，有兴趣的朋友可研究下文提到的几个开源库。

类加载器（class loader）用来加载 Java 类到 Java 虚拟机中。一般来说，Java 虚拟机使用 Java 类的方式如下：Java 源程序（.java 文件）在经过 Java 编译器编译之后就被转换成 Java 字节代码（.class 文件）。类加载器负责读取 Java 字节代码，并转换成java.lang.Class类的一个实例。Android 的 DVM(Dalvik Virtual Machine) 中 .class 文件还会进一步转换成 .dex 文件供 DVM 的 PathClassLoader 加载器加载。

Android 中类的加载器主要有:
1. DexClassLoader
可以加载包含 dex 的 jar 和 apk 文件。加载的文件可放置于sdcard目录下，但不要这么做，无权限控制的 sdcard 分区的 dex 文件易被破坏或代码注入攻击。
1. PathClassLoader。
Android 本身使用的系统加载器，用于加载 App 本身，只能加载已经安装到Android系统中的apk文件。

从上面可以看出，插件化的动态加载是使用 **DexClassLoader** 实现。下面简单的介绍如何实现类的动态加载。

## 动态加载实现
插件类是动态加载的，那么本地(主 App)便没有插件类的定义，要获取和运行插件类和方法有以下方法：
1. 反射
1. 继承接口或虚类

使用反射可获取插件类的全部方法和成员变量，但使用反射比较复杂而且需要硬编码方法名变量名等，使用继承公共接口的方式更加方便安全。
### 代码实现
下面的示例可在我的 github 中获取 [BlogDemo](https://github.com/HalfLike/BlogDemo)库，参考示例工程`dynamic-load-demo`。
这里先定义一个插件类的接口：

```
// ModuleInterface.java

public interface ModuleInterface {

    String print(String msg);

}
```

定义插件类实现 `ModuleInterface` 接口:

```
// QQModule.java
public class QQModule implements ModuleInterface {
    @Override
    public String print(String msg) {
        return "It is QQModule. " + msg;
    }
}
```

在 `MainActivity` 中动态加载插件类，并调用实现的方法:

```
// MainActivity.java
public class MainActivity extends Activity {
    static final String TAG = "dynamic";
    ModuleInterface mModule = null;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button btn = (Button) findViewById(R.id.button);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mModule != null) {
                    // 插件类加载成功，调用 print 方法
                    Toast.makeText(getApplicationContext(), mModule.print("load succeed"), Toast.LENGTH_LONG).show();
                } else {
                    // 插件类未加载
                    Toast.makeText(getApplicationContext(), "load faild", Toast.LENGTH_LONG).show();
                }
            }
        });
        loadModule();
    }

    void logger(String msg) {
        Log.d(TAG, msg);
    }

    // 加载插件类
    public void loadModule() {
        // 插件类的路径，这里放置于 sdcard 目录下，真正应用时应置于 /data/data 中应用的私有目录下
        String dexPath = Environment.getExternalStorageDirectory() + File.separator +  "output.jar";
        // 优化后 dex 文件的存放路径
        File optimizedDir = this.getCacheDir();
        if (! new File(dexPath).exists()) {
            logger("dexFile is no exits");
            return;
        }

        DexClassLoader dexClassLoader = new DexClassLoader(dexPath,
                optimizedDir.getAbsolutePath(), null, this.getClassLoader());
        try {
            // 加载插件类
            Class<?> module = dexClassLoader.loadClass("com.halflike.module.QQModule");
            // 获取插件类的对象，并转换为 ModuleInterface 类型
            mModule = (ModuleInterface) module.newInstance();
            logger(mModule.print("load success"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

其中 `DexClassLoader` 构造器定义为
> public DexClassLoader (String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent)

其中，

* `dexPath` 
为包含需要加载的类的 jav/apk 文件的路径。
* `optimizedDirectory` 
为经过系统优化后的 dex 文件的存放路径，不可为 null。不可放于sdcard分区中，避免代码注入攻击。
* `libraryPath` 
为包含 native librarys 的文件路径，可为 null。我们并未用到 native 库，因此置为 null。
* `parent` 
为生成的 DexClassLoader 的父类加载器。类的加载器遵循父类委托机制(又称代理模式)，即先让父类加载器试图加载该类，只有父类加载器无法加载该类时才尝试从自己的类路径中加载该类。因此这里使用 MainActivity 的加载器，避免加载器不同导致类的隔离而引发转换类型出错。更多类加载器的知道可参考[深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)

### 插件类的编译
先将 QQModule.java 编译成 .class 文件，注意需要将 ModuleInterface.java 接口文件也一起编译，否则会报错。在终端或命令行中运行 javac 如:

```
javac QQModule.java ModuleInterface.java
```

此时会生成两个 .class 文件，其中 ModuleInterface.class 可删除。用 jar 命令将 QQModule.class 打包。

```
jar cvf input.jar QQModule.class
```

再用 Android sdk 自带的工具 dx (位于 sdk/build-tools/版本号)打包出包含 classes.dex 文件的 output.jar。

```
dx --dex --no-strict --output=output.jar input.jar
```

使用 **--no-strict** 参数可避免文件结构和包名不一致而产生的解析错误：
> 
UNEXPECTED TOP-LEVEL EXCEPTION:
com.android.dx.cf.iface.ParseException: class name (com/halflike/module/QQModule) does not match path (QQModule.class)

将生成的 output.jar 放置到 sdcard 根目录下：

```
adb push output.jar /sdcard/
```

### 运行效果
将 QQModule.java 文件从工程中删除，运行Demo，从日志中可以看出加载成功，这时点 "SECOND DEX" 按钮会运行动态加载的类的方法，如图示：

## 动态更新
从之前的讨论可以知道，动态加载的插件类是在程序运行后调用加载方法后才加载的。在加载类前先从网络中下载准备好的 jar 包并加载即可达到动态更新的目的。这里只是讨论基本原理，实际应用了还要考虑用 md5 比较保证待加载包的完整性，是否最新，加载时机，加解密等问题。

## 开源项目
### [dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)
这个项目实现了一部分的动态加载，原理是 DexClassLoader 加 Activity 代理。

### [AndroidDynamicLoader](https://github.com/mmin18/AndroidDynamicLoader)
和上面不同的是：他不是用代理 Activity 的方式实现而是用 Fragment 以及 schema 的方式实现。

### [android-pluginmgr](https://github.com/houkx/android-pluginmgr/)
dynamic-load-apk和AndroidDynamicLoader都有一个共同点，需要对插件做一定的约束。
按照道理说，由于系统的限制这是非常合情合理的。但据说 android-pluginmgr 这个开源项目不需要对插件做任何限制，可直接运行插件的 Activity。我还没有进行验证，感兴趣的朋友可以先看下，欢迎一起交流。

## 参考：
1. 深入探讨 Java 类加载器 : https://www.ibm.com/developerworks/cn/java/j-lo-classloader/
1. Building Apps with Over 65K Methods : https://developer.android.com/tools/building/multidex.html
1. Android用DexClassLoader实现动态调用jar包 : http://blog.csdn.net/cheligeer1988/article/details/13774271
1. android-pluginmgr不需要插件规范的apk动态加载框架 : http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1230/2232.html