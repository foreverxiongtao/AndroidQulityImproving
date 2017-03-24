# App性能优化 #
##一.Android优化导图

![](http://i.imgur.com/W5loi13.png)


## 二.内存性能分析优化 ##

1.1.1 前言

了解内存优化之前，要了解一下java内存的分配，常见的内存分配为三种:

- 静态储存区(方法区)：编译时就分配好，在程序整个运行期间都存在。它主要存放静态数据和常量。
- 栈区：当方法执行时，会在栈区内存中创建方法体内部的局部变量，方法结束后自动释放内存,栈内存分配运算内置于处理器的指令集中(CPU高速缓存中)，效率很高，但是分配的内存容量有限。
- 堆区(动态内存)：通常存放new或者malloc出来的内存，内存大小是任意的。程序员自己手动去free(java中主要是靠垃圾回收器回收）。

1.1.2 GC机制(不可见)

GC 是 garbage collection 的缩写, 垃圾回收的意思. 也可以是 Garbage Collector, 也就是垃圾回收器.
垃圾回收器有三大职责:

1. 分配内存;
1. 确保任何被引用的对象保留在内存中;
1. 回收不能通过引用关系找到的对象的内存.


对于不同的语言平台来说，进行标记回收内存的算法是不一样的，像 Android（Java）则采用 GC-Root 的标记回收算法。下面这张图就展示了 Android 内存的回收管理策略（图来自Google 2011的IO大会）

![](http://i.imgur.com/eYmuLT4.png)

那GC回收对象的依据是什么呢 ?简单的说，对于一个对象，若果不存 在从 GC 根节点到该对象的引用链 (从根节点不可到达的 （从根节点不可到达的），那么对于 GC 来说这个对象就是需要被回收的，反之该对象是从根节点可到达的，那么这个对象就不会被 GC 回 收。

GC回收的一般流程是:

![](http://i.imgur.com/xRpxaSt.png)


1.1.3对象引用

Java的引用类型可以分为以下几种： 

- 强引用（Strong Ref）：强可达，去掉强可达，才会被回收。 
- 软引用（Soft Ref）：内存够用，就保持，内存吃紧，则回收，主要用来做缓存。 
- 弱引用（Weak Ref）：比Soft Ref弱，即使内存不吃紧也会被回收。 
- 虚引用（Phantom Ref）：不会在内存保持任何对象。

一图胜千言

![](http://i.imgur.com/rgpQrgW.png)

1.1.3 内存管理

Android是一个多任务系统，Android中每个App默认情况下是运行在一个独立进程中的, 而这个独立进程正是从Zygote孵化出来的VM进程. 也就是说, 每个App是运行在独立的VM空间的。讲到vm,我们就要看一下Android目前存在的VM.

(1)Dalvik & ART

Android在4.4之前一直使用的Dalvik虚拟机作为App的运行VM的, 4.4中引入了ART作为开发者备选, 5.0起正式将ART作为默认VM了.

Dalvik: 采用的是JIT(即时编译)技术, 在应用程序启动时, JIT通过进行连续的性能分析来优化程序代码的执行, 在程序运行的过程中, Dalvik在不断的进行将字节码编译成机器码的工作.

ART(Android Runtime):Android用其取代Dalvik,采用了AOT(ahead of time)预编译技术, 主要目的就是为了提升运行性能. 所以, ART相比Dalvik有几个关键的提升:在安装apk的过程中, ART会使用dex2oat程序所有的字节码预编译成了机器码. 应用程序运行过程中无需进行实时的编译工作, 只需要进行直接调用. 故而提高了应用程序的运行效率.

GC效率

- 由原来的两次GC暂停减少为一次.
- 以较少的GC时间回收最近分配的, 短命的对象.
- 提升GC工程学, 使并发GC更及时.
- 压缩GC, 以减少后台内存使用和内存碎片.


ART和Dalvik都是使用paging和memory-mapping(mmapping)来管理内存的. 这就意味着, 任何被分配的内存都会持续存在, 唯一的释放这块内存的方式就是释放对象引用(让对象GC Root不可达), 故而让GC程序来回收内存.

1.1.4 切换App时的内存管理机制

当用户切换App时, 被切换到后台的App所使用的内存并未因此删除, 该App进程被缓存到一个LRU缓存中, 以便用户切换回来时, 能更快的启动App, 让多任务更流畅.

但是, 当系统内存不够用的时候, 就会根据LRU特性, 以及上面说到的进程级别(同时也会考虑该App进程占用的内存大小)来决定杀死哪些进程, 来回收内存, 以便执行当前任务.

所以, 如果我们为了不要让系统kill掉我们的App, 可以从进程级别, 内存消耗量等几个方面进行优化.

1.1.5 App的内存限制

Android是一个多任务系统, 为了保证多任务的运行, Android给每个App可使用的Heap大小设定了一个限定值.这个值是系统设置的prop值, 系统编译时内置的, 保存在system/build.prop中. 一般国内的手机厂商都会做修改, 根据手机配置不同而不同。

![](http://i.imgur.com/lD49LY1.png)



### 1.内存泄漏###
内存泄漏是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。内存泄露有时不严重且不易察觉，这样开发者就不知道存在内存泄露，但有时也会很严重，会提示你Out of memory。

Android中内存泄漏的根本原因是什么呢？长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是因为长生命周期持有它的引用而导致不能被回收。
##### a.Activity对象未被回收 #####

   1.1静态变量引用Activty对象

静态变量引用Activty对象时，会导致Activty对象所占内存内漏。主要是因为，静态变量是驻扎在JVM的方法区，因此，静态变量引用的对象是不会被GC回收的，因为它们所引用的对象本身就是GC。即最终导致Activity对象不被回收，从而也就造成内存泄漏。
比如我们经常定义一个工具类，

    public class AppSettings {
    private Context mAppContext;
    private static AppSettings sInstance = new AppSettings();

    //some other codes
    public static AppSettings getInstance() {
      return sInstance;
    }
  
    public final void setup(Context context) {
        mAppContext = context;
    }
    }
sInstance作为静态对象，其生命周期要长于普通的对象，其中也包含Activity，当我们进行屏幕旋转，默认情况下，系统会销毁当前Activity，然后当前的Activity被一个单例持有，导致垃圾回收器无法进行回收，进而产生了内存泄露。

解决方案:不支持有Activity的引用，而是持有Application的Context的引用


1.2 静态View

Activity经常启动，View读取非常耗时，我们可以通过静态View变量来保持对该Activity的rootView引用。这样就可以不用每次启动Activity都去读取并渲染View了.但是一旦View attach到我们的Window上，就会持有一个Context(即Activity)的引用。所以如果使用静态View时，需要确保在资源回收时，将静态View detach掉。

1.3匿名类
 
匿名类也会持有外部类的引用
   
     @Override
    protected void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
         setContentView(R.layout.activity_main);
         new AsyncTask<Void, Void, Void>() {
        @Override
        protected Void doInBackground(Void... params) {
            //该线程中持有Activity的引用，并且不释放
             while (true) ;
         }
    }.execute();
   		}
    }
解决方案：第一，在Activity生命周期结束前，去cancel AsyncTask，因为Activity都要销毁了，这个时候再跑线程，绘UI显然已经没什么意义了。第二，如果一定要写成内部类的形式，对context采用WeakRefrence,在使用之前判断是否为空。

1.4 内部类

同匿名类一样,非静态的内部类会持有外部类的一个引用，因此，如果我们在一个外部类中定义一个静态变量，这个静态变量是引用内部类对象。将会导致内存泄漏！因为这相当于间接导致静态引用外部类。
    
    private static InnerClass mInnerClass;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    	super.onCreate(savedInstanceState);
    	setContentView(R.layout.activity_main); 
    	innerClass = this.new InnerClass(); 
    }
    class InnerClass(){
    
      }
    }
    
解决方案:可以将内部类定义成静态内部类，因为静态内部类的创建和嵌套类(外部类)无关，不持有外部类的引用，同时通过弱引用的方式来对context采用WeakRefrence,在使用之前判断是否为空。

1.5 handler内存泄漏

我们知道，主线程的Looper对象不断从消息队列中取出消息，然后再交给Handler处理。如果在Activity中定义Handler对象，那么Handler肯定是持有Activty的引用。而每个Message对象是持有Handler的引用的（Message对象的target属性持有Handler引用），从而导致Message间接引用到了Activity。如果在Activty destroy之后，消息队列中还有Message对象，Activty是不会被回收的。当然了，如果消息正在准备（处于延时入队期间）放入到消息队列中也是一样的。

错误使用方式:

     private Handler handler = new Handler() {
        public void handleMessage(android.os.Message msg) {
            if (msg.what == 1) {
                mAdapter.notifyDataSetChanged();
            }
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    	super.onCreate(savedInstanceState);
    	setContentView(R.layout.activity_main); 
       }
    }
 

正确使用方式:
    static class MyHandler extends Handler {
        WeakReference<Activity> mWeakReference;

        public MyHandler(Activity activity) {
            mWeakReference = new WeakReference<Activity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            final Activity activity = mWeakReference.get();
            if (activity != null) {
                if (msg.what == 1) {
                    mAdapter.notifyDataSetChanged();
                }
            }
        }
    }
另外你的Handler是被delay的Message持有了引用，那么使用相应的Handler的removeCallbacks()方法，把消息对象从消息队列移除就行了。

1.6 Threads，TimerTask，异步线程持有匿名的内部类之类

    void spawnThread() {
    new Thread() {
    @Override public void run() {
   		   while(true);
    	  }
    	}.start();
    }
    
    void scheduleTimer() {
    new Timer().schedule(new TimerTask() {
    @Override
    public void run() {
          while(true);
        }
      }, Long.MAX_VALUE >> 1);
    }

       Runnable ref2 = new Runnable() {
        @Override
        public void run() {

        }
    };



1.7WebView造成的泄露（IOS使用WKWebview代替UIWebView,android呢）

当我们不要使用WebView对象时，应该调用它的destory()函数来销毁它，并释放其占用的内存，否则其占用的内存长期也不能被回收，从而造成内存泄露。

1.让 WebView 独立运行在一个进程里，用完 WebView 后直接销毁这个进程，即使内存泄露了，也不会影响到主进程。微信，手 Q 等 App 也采用了这个方案。但是这就涉及到了跨进程通讯，处理起来就比较麻烦。

2.另外个解决方案，就是使用自己封装的 WebView，比如上面提到的 X5 内核，且使用 WebView 的时候，不在 XML 里面声明，而是在代码中直接 new 出来，传入 application context 来防止 activity 引用被滥用。

3.使用第三方 WebView 内核
X5 内核
212 KB

Web页面crash率降低75%

页面打开速度提升35%

流量节省60%

U3内核


在使用了这个方式后，基本上 90% 的 WebView 内存泄漏的问题便得以解决。


1.8 Dialog
Dialog导致Window泄露,如果需要在dialog依附的Activity销毁前没有调用dialog.dismiss().会导致Activity泄露

1.8监听注册后没有反注册

 1.broadCastReceiver注册后，没有反注册

       /**
    	 * 注册广播
	 */
	private void register()
	{
        IntentFilter filter = new IntentFilter();  
        filter.addAction("your action............");   
        registerReceiver(mReceiver, filter);  
	}
	
	/**
	 * 反注册广播
	 */
	private void unRegister()
	{
		 unregisterReceiver(mReceiver);  
	}
	
	/**
	 * 广播
	 */
	private BroadcastReceiver mReceiver=new BroadcastReceiver()
	{
		@Override
		public void onReceive(Context context, Intent intent) {
            
		}
	};
2.service通过绑定的方式开启后，在结束的时候没有进行解绑.

3.第三方框架比如eventBus之类的在页面destroy的时候需要记得解绑

4.各种callBack/Listener的泄露，要及时设置为Null，特别是static的callback

### 切记:在使用一些框架或者服务之类的会出现，add/remove, register/unregister，subscriber/unsubscrib, bind/unbin之类的和当前的页面进行关联，碰到这种类似的字眼，一定要比较敏感，因为有可能需要我们在页面关闭的时候手动的去解绑 ###

##### b.资源未关闭造成的内存泄漏 ####
对于使用了BraodcastReceiver，ContentObserver，File，Cursor，Stream，Bitmap、网络连接等资源的使用，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，造成内存泄漏。


##### c.集合对象造成的泄漏 ###
当我们定义一个静态的集合类时，请注意，这可能会导致内存泄漏！前面我们提到过，静态变量所引用的对象是不会被回收掉的。而我的静态集合类中，包含有大量的对象，这些对象不会被回收。另外，如果集合中保存的对象又引用到了其他的大对象，如超长字符串、Bitmap、大数组等，很容易造成OOM。

private List<EmotionPanelInfo> data;    
public void onDestory() {        
    if (data != null) {
        data.clear();
        data = null;
    }
}


### 解决内存泄漏工具 ##

- 1.Square公司的leakcanary
- 2.Memory Monitor(As自带)
- 3.静态代码分析工具 —— Lint
- 4.FindBugs 、 Checkstyle(As插件)
- 5.StrictMode

### 2.避免内存消耗过大
2.1.Bitmap

当你加载Bitmap 的时候，需要在屏幕上显示的时候才将它加载到内存里，或者通过缩放原图的尺寸来减小内存占用。请记住随着 Bitmap 尺寸的增长，图片所消耗的内存会成平方量级的增长，因为 Bitmap 的 x 轴和 y 轴都在增长.
注:尽量避免手动去处理Bitmap，Glide，Fresco等图片加载框架都有更完善的方式去处理，去节省内存开销

推荐一个图片压缩网站:https://tinypng.com/，大部分情况下可以节省至少60%以上，并且是基本上损失比较小

2.2 合理使用Service

   Service的及时关闭可以让我们节省内存消耗, 对于一次性的任务, 建议使用IntentService.

2.3使用优化后的数据容器

使用Android提供的SparseArray, SparseBooleanArray, LongSparseArray来代替HashMap的使用.

2.4少用枚举enum结构

相比于静态常量(static final), enum会耗费双倍的内存.

2.5 警惕抽象代码
   抽象代码可以提升代码的灵活性和可维护性，抽象方法可能带来很多的额外花费，例如当他们执行的时候，他们拥有大量的代码，并且他们会被多次映射到内存中占用更多的内存，因此如果抽象的效果不是很好，那么最好放弃他

2.6采用Parcel机制
  常使用Parcel类的场景就是在Activity间传递数据。在Activity间使用Intent传递数据的时候，可以通过Parcelable机制传递复杂的对象，parcel对象在内存中序列化和反序列话的时候利用的连续的内存空间

2.7当界面不可见时释放内存

 当用户打开了另外一个程序，我们的程序界面已经不可见的时候，我们应当将所有和界面相关的资源进行释放。重写Activity的onTrimMemory()方法，然后在这个方法中监听TRIM_MEMORY_UI_HIDDEN这个级别，一旦触发说明用户离开了程序，此时就可以进行资源释放操作了。

2.8其它

- Java 中的每一个类（包括匿名内部类），都会消耗大约500比特内存
- 每一个类对象都会消耗12-16比特内存
- 把单个 Entry 放入 HashMap 需要多消耗32比特的内存


### 3.内存抖动 ##
解释:因为大量的对象被创建又在短时间内马上被释放。内存抖动导致频繁的GC则会显著占用帧间隔时间(16ms),从而导致页面无法渲染，出现丢帧，页面卡顿。
for,while这种基于循环的一定要注意

3.1使用对象池避免频繁创建对象(享元模式)

    public class MyObject {
    private static final Pools.SynchronizedPool<MyObject> MY_POOLS = new Pools.SynchronizedPool<>(10);

    public static MyObject obtain() {
        MyObject object = MY_POOLS.acquire();
        if (object == null)
            object = new MyObject();
        return object;
    }

    public void recycle() {
        MY_POOLS.release(this);
    }

![](http://i.imgur.com/qhCEnyg.png)


## 三.UI布局优化 ##

### 1.Android渲染机制###
Android系统每隔16ms就会发出一个VSYNC信号,从而触发对UI的一次渲染，如果整个过程控制在16ms以内，就能达到流畅的画面，如果你的某个操作花费时间是24ms，系统在得到VSYNC信号的时候就无法进行正常渲染，这样就发生了丢帧现象。那么用户在操作时间内看到的会是同一帧画面。那么就会出现丢帧，从而给用户带来卡顿的效果
正常的页面:

![](http://i.imgur.com/51jqDZY.png)

卡顿丢帧后的页面:

![](http://i.imgur.com/wLCopxO.png)

出现丢帧的原因:
- layout太过复杂
- UI上有层叠太多的绘制单元
- 动画执行 的次数过多


对应的优化方案:

- 尽量不要嵌套使用LinearLayout(嵌套过多建议使用RelativeLayout).
- 尽量不要在嵌套的LinearLayout中都使用weight属性.（子组件会被测量两次）
- Layout的选择, 以尽量减少View树的层级为主.
- 去除不必要的父布局.
- 善用TextView的Drawable减少布局层级
- 使用include来重用布局.
- 使用<merge>来解决include或自定义组合ViewGroup导致的冗余层级问题
- 如果Hierarchy Viewer查看层级超过5层, 你就需要考虑优化下布局了
- 使用ViewStub来进行延迟加载，节约内存并可以提高界面渲染速度
- onDraw方法避免执行大量的操作.

---onDraw中不要创建大量的局部对象，因为onDraw方法会被频繁调用，这样就会在一瞬间产生大量的临时对象，不仅会占用过多内存还会导致系统频繁GC，降低程序执行效率。

---onDraw也不要做耗时的任务，也不能执行成千上万的循环操作，尽管每次循环都很轻量级，但大量循环依然十分抢占CPU的时间片，这会造成View的绘制过程不流畅

- listView的一些优化...不说了 最好使用Recyclerview替换listView(局部刷新，配合glide，异步任务)
- viewpager的延迟加载策略
- 8.臭名昭著的ANR；
- 减少垃圾回收器的GC次数。


检测工具:
1.Hierarchy Viewer减少层级
Window Tree View Tree OverView  Layout View 四个部分
![](http://i.imgur.com/uGLlA7f.png)

![](http://i.imgur.com/to2AOiq.png)

![](http://i.imgur.com/JdPo7Be.png)

2.Lint tool

![](http://i.imgur.com/VZADVWp.png)

![](http://i.imgur.com/H4NG8BT.png)



3.ui线程卡慢性能监控工具BlockCanary(Alibaba)
用来检测线程，IO,计算等阻塞

![](http://i.imgur.com/NZRK3DN.png)

![](http://i.imgur.com/a6FdAyR.png)

2.重绘工具 GPU Overdraw 

本色 —— 没有发生过度绘制 —— 屏幕上的像素点只被绘制了 1 次。

蓝色 —— 1 倍过度绘制 —— 屏幕上的像素点被绘制了 2 次。

绿色 —— 2 倍过度绘制 —— 屏幕上的像素点被绘制了 3 次。

粉色 —— 3 倍过度绘制 —— 屏幕上的像素点被绘制了 4 次。

红色 —— 4 倍过度绘制 —— 屏幕上的像素点被绘制了 5 次。

![](http://i.imgur.com/Q1VTeWC.png)
 
![](http://i.imgur.com/vdQN0lM.png)

![](http://i.imgur.com/hLUbhFl.png)

3.Android Monitor(GPU,CPU,NETWORK,MEMERY)

去掉app主题，去除不必要的背景色。
友情链接:谷歌官方优化方案:Android性能优化典范
1.[http://www.csdn.net/article/2015-01-20/2823621-android-performance-patterns](http://www.csdn.net/article/2015-01-20/2823621-android-performance-patterns)

2.
[https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE "Android性能优化专题(2015年初发布)")



## 四.网络性能优化 ##

Retrofit是目前最好的网络请求框架。官网地址:[http://square.github.io/retrofit/](http://square.github.io/retrofit/)

替代异步任务最好的东西是RxJava，没有之一。官网地址:[http://gank.io/post/560e15be2dca930e00da1083](http://gank.io/post/560e15be2dca930e00da1083)

## 五.编码优化 ##

（1）避免创建不必要的对象


- 如果我们有一个需要拼接的字符串，那么可以优先考虑使用StringBuffer或者StringBuilder来进行拼接，而不是加号连接符，因为使用加号连接符会创建多余的对象，拼接的字符串越长，加号连接符的性能越低。

- 在没有特殊原因的情况下，尽量使用基本数据类来代替封装数据类型，int比Integer要更加高效，其它数据类型也是一样。

- 当一个方法的返回值是String的时候，通常可以去判断一下这个String的作用是什么，如果我们明确地知道调用方会将这个返回的String再进行拼接操作的话，可以考虑返回一个StringBuffer对象来代替，因为这样可以将一个对象的引用进行返回，而返回String的话就是创建了一个短生命周期的临时对象。

- 正如前面所说，基本数据类型要优于对象数据类型，类似地，基本数据类型的数组也要优于对象数据类型的数组。另外，两个平行的数组要比一个封装好的对象数组更加高效，举个例子，Foo[]和Bar[]这样的两个数组，使用起来要比Custom(Foo,Bar)[]这样的一个数组高效得多。 

- 对常量使用static final修饰符(所有的常量都会在dex文件的初始化器当中进行初始化,定义类就不需要<clinit>方法了。当我们调用intVal时可以直接指向42的值，而调用strVal时会用一种相对轻量级的字符串常量方式，而不是字段搜寻的方式)

- 多用系统的API

- 避免在内部调用Getters/Setters方法

 高性能编码方面《Efficient Java》,这本书大家可以有兴趣可以查阅


## 六.其他的优化 ##

- App整体的架构
 
- 使用依赖注入能让你的app更模块化更好测试。

- 导入任何第三方包的时候都要再三思考，因为这个动作责任重大。

- 使用 Android Studio -> Analyze 去分析和定位bug 

- 分包按功能特点分，不要按业务层分 

- 命名规范 推荐:[http://mp.weixin.qq.com/s/Xn7fM4zsNrvpDW8xW2m_hQ](http://mp.weixin.qq.com/s/Xn7fM4zsNrvpDW8xW2m_hQ)

    .........
##写在最后

有人的地方就有江湖、有代码的地方就有优化，代码不止、优化不止......





