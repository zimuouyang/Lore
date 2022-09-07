(57 条消息) Android Jetpack 系列之 Lifecycle_- 小马快跑 - 的博客 - CSDN 博客







### 文章目录

- - [Lifecycle 介绍](https://blog.csdn.net/u013700502/article/details/118469311#Lifecycle_2)
  - [场景 case](https://blog.csdn.net/u013700502/article/details/118469311#case_12)
  - [Lifecycle 使用](https://blog.csdn.net/u013700502/article/details/118469311#Lifecycle_57)
  - - [Activity/Fragment 中使用 Lifecycle](https://blog.csdn.net/u013700502/article/details/118469311#ActivityFragmentLifecycle_109)
    - [自定义 LifecycleOwner](https://blog.csdn.net/u013700502/article/details/118469311#LifecycleOwner_194)
    - [Application 中使用 Lifecycle](https://blog.csdn.net/u013700502/article/details/118469311#ApplicationLifecycle_296)
    - [Service 中使用 Lifecycle](https://blog.csdn.net/u013700502/article/details/118469311#ServiceLifecycle_349)
    - [完整代码地址](https://blog.csdn.net/u013700502/article/details/118469311#_410)
  - [源码解析](https://blog.csdn.net/u013700502/article/details/118469311#_413)
  - - [Lifecycle.java](https://blog.csdn.net/u013700502/article/details/118469311#Lifecyclejava_414)
    - [Event 生命周期事件分发 & 接收](https://blog.csdn.net/u013700502/article/details/118469311#Event_483)
  - [参考](https://blog.csdn.net/u013700502/article/details/118469311#_1064)





## Lifecycle 介绍



`Lifecycle`可以让某一个类变成`Activity`、`Fragment`的[生命周期](https://so.csdn.net/so/search?q=生命周期&spm=1001.2101.3001.7020)观察者类，监听其生命周期的变化并可以做出响应。`Lifecycle`使得代码更有条理性、精简、易于维护。



Lifecycle 中主要有两个角色：



- LifecycleOwner: 生命周期拥有者，如 Activity/Fragment 等类都实现了该接口并通过 getLifecycle() 获得 Lifecycle，进而可通过 addObserver() 添加观察者。
- LifecycleObserver: 生命周期观察者，实现该接口后就可以添加到 Lifecycle 中，从而在被观察者类生命周期发生改变时能马上收到通知。



实现`LifecycleOwner`的生命周期拥有者可与实现`LifecycleObserver`的观察者完美配合。



## 场景 case



假设我们有一个在屏幕上显示设备位置的 Activity，我们可能会像下面这样实现：



```
internal class MyLocationListener(
        private val context: Context,
        private val callback: (Location) -> Unit) {

    fun start() {
        // connect to system location service
    }

    fun stop() {
        // disconnect from system location service
    }
}

class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this) { location ->
            // update UI
        }
    }

    public override fun onStart() {
        super.onStart()
        myLocationListener.start()
        // manage other components that need to respond
        // to the activity lifecycle
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
        // manage other components that need to respond
        // to the activity lifecycle
    }
}
```



`注：上面代码来自官方示例`~



我们可以在`Activity` 或 `Fragment` 的生命周期方法 (示例中的`onStart/onStop`) 中直接对依赖组件进行操作。但是，这样会导致代码条理性很差且不易扩展。那么有了`Lifecycle`，可以将依赖组件的代码从`Activity/Fragment`生命周期方法中移入组件本身中。



## Lifecycle 使用



根目录下 build.gradle:



```
allprojects {
    repositories {
        google()

        // Gradle小于4.1时，使用下面的声明替换:
        // maven {
        //     url 'https://maven.google.com'
        // }
        // An alternative URL is 'https://dl.google.com/dl/android/maven2/'
    }
}
```



app 下的 build.gradle:



```
    dependencies {
        def lifecycle_version = "2.3.1"
        def arch_version = "2.1.0"

        // ViewModel
        implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$lifecycle_version"
        // LiveData
        implementation "androidx.lifecycle:lifecycle-livedata-ktx:$lifecycle_version"
        // Lifecycles only (without ViewModel or LiveData)
        implementation "androidx.lifecycle:lifecycle-runtime-ktx:$lifecycle_version"

        // Saved state module for ViewModel
        implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:$lifecycle_version"

        // Annotation processor
        kapt "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
        // 可选 - 如果使用Java8,使用下面这个代替lifecycle-compiler
        implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"

        // 可选 - 在Service中使用Lifecycle
        implementation "androidx.lifecycle:lifecycle-service:$lifecycle_version"

        // 可选 - Application中使用Lifecycle
        implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version"

        // 可选 - ReactiveStreams support for LiveData
        implementation "androidx.lifecycle:lifecycle-reactivestreams-ktx:$lifecycle_version"

        // 可选 - Test helpers for LiveData
        testImplementation "androidx.arch.core:core-testing:$arch_version"
    }
```



### Activity/[Fragment](https://so.csdn.net/so/search?q=Fragment&spm=1001.2101.3001.7020) 中使用 Lifecycle



首先先来实现`LifecycleObserver`观察者：



```
open class MyLifeCycleObserver : LifecycleObserver {

    @OnLifecycleEvent(value = Lifecycle.Event.ON_START)
    fun connect(owner: LifecycleOwner) {
        Log.e(JConsts.LIFE_TAG, "Lifecycle.Event.ON_CREATE:connect")
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_STOP)
    fun disConnect() {
        Log.e(JConsts.LIFE_TAG, "Lifecycle.Event.ON_DESTROY:disConnect")
    }
}
```



在方法上，我们使用了`@OnLifecycleEvent`注解，并传入了一种生命周期事件，其类型可以为`ON_CREATE`、`ON_START`、`ON_RESUME`、`ON_PAUSE`、`ON_STOP`、`ON_DESTROY`、`ON_ANY`中的一种。其中前 6 个对应 Activity 中对应生命周期的回调，最后一个 ON_ANY 可以匹配任何生命周期回调。
所以，上述代码中的`connect()、disConnect()`方法分别应该在`Activity`的`onStart()、onStop()`中触发时执行。接着来实现我们的`Activity`:



```
class MainActivity : AppCompatActivity() {

      override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onCreate")

        //添加LifecycleObserver观察者
        lifecycle.addObserver(MyLifeCycleObserver())
    }

    override fun onStart() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onStart")
        super.onStart()
    }

    override fun onResume() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onResume")
        super.onResume()
    }

    override fun onPause() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onPause")
        super.onPause()
    }

    override fun onStop() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onStop")
        super.onStop()
    }

    override fun onDestroy() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onDestroy")
        super.onDestroy()
    }
}
```



可以看到在 Activity 中我们只是在 onCreate() 中添加了这么一行代码：



```
lifecycle.addObserver(MyLifeCycleObserver())
```



其中`getLifecycle()`是`LifecycleOwner`中的方法，返回的是`Lifecycle`对象，并通过`addObserver()`的方式添加了我们的生命周期观察者。接下来看执行结果，启动`Activity`:



```
2021-06-30 20:57:58.038 11257-11257/ E/Lifecycle_Study: ACTIVITY:onCreate

//onStart() 传递到 MyLifeCycleObserver: connect()
2021-06-30 20:57:58.048 11257-11257/ E/Lifecycle_Study: ACTIVITY:onStart
2021-06-30 20:57:58.049 11257-11257/ E/Lifecycle_Study: Lifecycle.Event.ON_START:connect

2021-06-30 20:57:58.057 11257-11257/ E/Lifecycle_Study: ACTIVITY:onResume
```



关闭 Activity:



```
2021-06-30 20:58:02.646 11257-11257/ E/Lifecycle_Study: ACTIVITY:onPause

//onStop() 传递到 MyLifeCycleObserver: disConnect()
2021-06-30 20:58:03.149 11257-11257/ E/Lifecycle_Study: ACTIVITY:onStop
2021-06-30 20:58:03.161 11257-11257/ E/Lifecycle_Study: Lifecycle.Event.ON_STOP:disConnect

2021-06-30 20:58:03.169 11257-11257/ E/Lifecycle_Study: ACTIVITY:onDestroy
```



可以看到我们的`MyLifeCycleObserver`中的`connect()/disconnect()`方法的确是分别在`Activity`的`onStart()/onStop()`回调时执行的。



### 自定义 LifecycleOwner



在`AndroidX`中的`Activity、Fragmen`实现了`LifecycleOwner`，通过`getLifecycle()`能获取到`Lifecycle`实例 (`Lifecycle`是抽象类，实例化的是子类`LifecycleRegistry`)。



```
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}

public class LifecycleRegistry extends Lifecycle {

}
```



如果我们想让一个自定义类成为`LifecycleOwner`，可以直接实现`LifecycleOwner`：



```
class CustomLifeCycleOwner : LifecycleOwner {
    private lateinit var registry: LifecycleRegistry

    fun init() {
        registry = LifecycleRegistry(this)
        //通过setCurrentState来完成生命周期的传递
        registry.currentState = Lifecycle.State.CREATED
    }

    fun onStart() {
        registry.currentState = Lifecycle.State.STARTED
    }

    fun onResume() {
        registry.currentState = Lifecycle.State.RESUMED
    }

    fun onPause() {
        registry.currentState = Lifecycle.State.STARTED
    }

    fun onStop() {
        registry.currentState = Lifecycle.State.CREATED
    }

    fun onDestroy() {
        registry.currentState = Lifecycle.State.DESTROYED
    }

    override fun getLifecycle(): Lifecycle {
        //返回LifecycleRegistry实例
        return registry
    }
}
```



首先我们的自定义类实现了接口`LifecycleOwner`，并在`getLifecycle()`返回`LifecycleRegistry`实例，接下来就可以通过`LifecycleRegistry#setCurrentState`来传递生命周期状态了。到目前为止，已经完成了大部分工作，最后也是需要去添加`LifecycleObserver`:



```
//注意：这里继承的是Activity，本身并不具备LifecycleOwner能力
class MainActivity : Activity() {
    val owner: CustomLifeCycleOwner = CustomLifeCycleOwner()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onCreate")

        //自定义LifecycleOwner
        owner.init()
        //添加LifecycleObserver
        owner.lifecycle.addObserver(MyLifeCycleObserver())
    }

    override fun onStart() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onStart")
        super.onStart()
        owner.onStart()
    }

    override fun onResume() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onResume")
        super.onResume()
        owner.onResume()
    }

    override fun onPause() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onPause")
        super.onPause()
        owner.onPause()
    }

    override fun onStop() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onStop")
        super.onStop()
        owner.onStop()
    }

    override fun onDestroy() {
        Log.e(JConsts.LIFE_TAG, "$ACTIVITY:onDestroy")
        super.onDestroy()
        owner.onDestroy()
    }
}
```



很简单，主要是在`onCreate()`里实例化`LifecycleOwner`并调用`init()`完成`LifecycleRegistry`实例化。接着跟`androidX`中的`Activity`一样了，通过`getLifecycle()`得到`LifecycleRegistry`实例并通过`addObserver()`注册`LifecycleObserver`，最后代码执行结果跟上面的结果一致，不再重复贴了。



### Application 中使用 Lifecycle



首先记得要先引入对应依赖：



```
implementation "androidx.lifecycle:lifecycle-process:$lifecycle_version"
```



然后代码编写如下：



```
//MyApplicationLifecycleObserver.kt
class MyApplicationLifecycleObserver : LifecycleObserver {

    @OnLifecycleEvent(value = Lifecycle.Event.ON_START)
    fun onAppForeground() {
        Log.e(JConsts.LIFE_APPLICATION_TAG, "onAppForeground")
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_STOP)
    fun onAppBackground() {
        Log.e(JConsts.LIFE_APPLICATION_TAG, "onAppBackground")
    }
}

//MyApplication.kt
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        ProcessLifecycleOwner.get().lifecycle.addObserver(MyApplicationLifecycleObserver())
    }
}

//manifest.xml
<application
   android:name=".MyApplication">
</application>
```



启动应用：



```
2021-06-30 21:55:11.657 14292-14292/ E/Lifecycle_App_Study: onAppForeground
```



点击 Home 键，应用切到后台：



```
2021-06-30 21:55:35.741 14292-14292/ E/Lifecycle_App_Study: onAppBackground
```



再切回来：



```
2021-06-30 21:55:11.657 14292-14292/ E/Lifecycle_App_Study: onAppForeground
```



`ProcessLifecycleOwner`可以很方便地帮助我们检测 App 前后台状态。



### Service 中使用 Lifecycle



在`Service`中使用`Lifecycle`同样需要先引入依赖：



```
implementation "androidx.lifecycle:lifecycle-service:$lifecycle_version"
```



然后继承`LifecycleService`：



```
//MyService.kt
class MyService : LifecycleService() {
    override fun onCreate() {
        Log.e(JConsts.SERVICE, "Service:onCreate")
        super.onCreate()
        lifecycle.addObserver(MyLifeCycleObserver())
    }

    override fun onStart(intent: Intent?, startId: Int) {
        Log.e(JConsts.SERVICE, "Service:onStart")
        super.onStart(intent, startId)
    }

    override fun onDestroy() {
        Log.e(JConsts.SERVICE, "Service:onDestroy")
        super.onDestroy()
    }
}

//MainActivity.kt
  /**
   * 启动Service
   */
  private fun startLifecycleService() {
      val intent = Intent(this, MyService::class.java)
      startService(intent)
  }

  /**
   * 关闭Service
   */
  fun closeService(view: View) {
      val intent = Intent(this, MyService::class.java)
      stopService(intent)
  }
```



`LifecycleService`中实现了`LifecycleOwner`接口，所以子类中可以直接通过`getLifecycle()`添加生命周期`Observer`，在`Activity`中启动`Service`:



```
2021-07-01 14:10:09.703 7606-7606/ E/SERVICE: Service:onCreate

2021-07-01 14:10:09.709 7606-7606/ E/SERVICE: Service:onStart
2021-07-01 14:10:09.712 7606-7606/ E/SERVICE: Lifecycle.Event.ON_START:connect
```



操作停止 Service：



```
2021-07-01 14:10:13.062 7606-7606/ E/SERVICE: Service:onDestroy
2021-07-01 14:10:13.063 7606-7606/ E/SERVICE: Lifecycle.Event.ON_STOP:disConnect
```



结果也很简单，这里注意一点：因为`Service`中没有`onPause/onStop`状态，所以在`Service#onDestroy()`之后，`LifecycleService` 里会同时分发`Lifecycle.Event.ON_STOP、Lifecycle.Event.ON_DESTROY`两个`Event`，所以我们的观察者[监听](https://so.csdn.net/so/search?q=监听&spm=1001.2101.3001.7020)哪个都是可以的。



### 完整代码地址



完整代码地址参见：[Jetpack Lifecycle 例子](https://github.com/crazyqiang/AndroidStudy/tree/master/app/src/main/java/org/ninetripods/mq/study/jetpack/lifecycle)



## [源码](https://so.csdn.net/so/search?q=源码&spm=1001.2101.3001.7020)解析



### Lifecycle.java



```
public abstract class Lifecycle {

    @NonNull
    AtomicReference<Object> mInternalScopeRef = new AtomicReference<>();

    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    @NonNull
    public abstract State getCurrentState();

    //生命周期事件 对应于Activity/Fragment生命周期
    public enum Event {

        ON_CREATE,

        ON_START,

        ON_RESUME,

        ON_PAUSE,

        ON_STOP,

        ON_DESTROY,
        /**
         * An constant that can be used to match all events.
         */
        ON_ANY
    }

    //生命周期状态
    public enum State {
        //onStop()之后，此状态是LifecycleOwner终态，Lifecycle不在分发Event
        DESTROYED,

        //初始化状态
        INITIALIZED,

        //onCreate()或onStop()之后
        CREATED,

        //onStart()或onPause()之后
        STARTED,

        //onResume()之后
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```



`Lifecycle`中的两个重要枚举：



- Event：生命周期事件，包括`ON_CREATE`、`ON_START`、`ON_RESUME`、`ON_PAUSE`、`ON_STOP`、`ON_DESTROY`、`ON_ANY`，对应于`Activity/Fragment`生命周期。
- State：生命周期状态，包括`DESTROYED`、`INITIALIZED`、`CREATED`、`STARTED`、`RESUMED`，`Event`的改变会使得`State`也发生改变。



两者关系如下：

![img](https://img-blog.csdnimg.cn/img_convert/98211a74ee02c51a29056d0e6d378d34.png)





### Event 生命周期事件分发 & 接收



我们的`Activity`继承自`AppCompatActivity`，继续往上找`AppCompatActivity`的父类，最终能找到了`ComponentActivity`：



```
public class ComponentActivity extends androidx.core.app.ComponentActivity implements LifecycleOwner{

//省略其他代码 只显示Lifecycle相关代码
private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //将生命周期的事件传递交给ReportFragment
        ReportFragment.injectIfNeededIn(this);
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }
}

    @CallSuper
    @Override
    protected void onSaveInstanceState(@NonNull Bundle outState) {
        Lifecycle lifecycle = getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).setCurrentState(Lifecycle.State.CREATED);
        }
        super.onSaveInstanceState(outState);
        mSavedStateRegistryController.performSave(outState);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```



可以看到`getLifecycle()`返回的是`LifecycleRegistry`实例，并且在`onSaveInstanceState()`中分发了`Lifecycle.State.CREATED`状态，但是其他生命周期回调中并没有写了呀，嗯哼？再细看一下，`onCreate()`中有个`ReportFragment.injectIfNeededIn(this)`，直接进去看看：



```
public class ReportFragment extends Fragment {
    private static final String REPORT_FRAGMENT_TAG = "androidx.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";

    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
         activity.registerActivityLifecycleCallbacks(
                    new LifecycleCallbacks());
        }
       
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }

    @SuppressWarnings("deprecation")
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        //已经被标注为@Deprecated
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(@NonNull Lifecycle.Event event) {
        if (Build.VERSION.SDK_INT < 29) {
            dispatch(getActivity(), event);
        }
    }

    //API29及以上直接使用Application.ActivityLifecycleCallbacks来监听生命周期
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
        @Override
        public void onActivityCreated(@NonNull Activity activity,
                @Nullable Bundle bundle) {
        }

        @Override
        public void onActivityPostCreated(@NonNull Activity activity,
                @Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }

        @Override
        public void onActivityStarted(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }

        @Override
        public void onActivityResumed(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPostResumed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_RESUME);
        }

        @Override
        public void onActivityPrePaused(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_PAUSE);
        }

        @Override
        public void onActivityPaused(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPreStopped(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_STOP);
        }

        @Override
        public void onActivityStopped(@NonNull Activity activity) {
        }

        @Override
        public void onActivitySaveInstanceState(@NonNull Activity activity,
                @NonNull Bundle bundle) {
        }

        @Override
        public void onActivityPreDestroyed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_DESTROY);
        }

        @Override
        public void onActivityDestroyed(@NonNull Activity activity) {
        }
    }
}
```



代码的逻辑很清晰，主要通过一个透明的`Fragment`来分发生命周期事件，这样对于 Activity 来说是无侵入的。分成两部分逻辑：当 API>=29 时，直接使用`Application.ActivityLifecycleCallbacks`来分发生命周期事件；而当 API<29 时，在 Fragment 的生命周期回调中进行了事件分发。但殊途同归，两者最终都会走到
`dispatch(Activity activity, Lifecycle.Event event)`方法中，该方法内部又调用了`LifecycleRegistry#handleLifecycleEvent(event)`，我们继续去看该方法的实现：



```
//LifecycleRegistry.java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}

private void moveToState(State next) {
    //如果与当前状态一致 直接返回
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```



`getStateAfter()`根据传入的`Event`返回`State`，比如`ON_CREATE、ON_STOP`之后对应的是`CREATED`，这里再把之前的这张图贴出来就一目了然了:

![img](https://img-blog.csdnimg.cn/img_convert/98211a74ee02c51a29056d0e6d378d34.png)


得到`state`后，传入了`moveToState()`中，方法内部做了一些校验判断，然后又走到了`sync()`中：





```
/**
 * Custom list that keeps observers and can handle removals / additions during traversal.
 *
 * Invariant: at any moment of time for observer1 & observer2:
 * if addition_order(observer1) < addition_order(observer2), then
 * state(observer1) >= state(observer2),
 */
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
        new FastSafeIterableMap<>();


private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}

private boolean isSynced() {
    if (mObserverMap.size() == 0) {
        return true;
    }
    State eldestObserverState = mObserverMap.eldest().getValue().mState;
    State newestObserverState = mObserverMap.newest().getValue().mState;
    //最新状态和最老状态一致 且 当前状态与最新状态一致(传进来的状态与队列中的状态一致) 两个条件都符合时，即认为是状态同步完
    return eldestObserverState == newestObserverState && mState == newestObserverState;
}

private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}

private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
```



`FastSafeIterableMap< LifecycleObserver, ObserverWithState>`实现了在遍历过程中的安全增删元素。`LifecycleObserver`是观察者，`ObserverWithState`则是对观察者的封装。`isSynced()`用来判断所有的观察者状态是否同步完，如果队列中新老状态不一致或者传进来的 State 与队列中的不一致，会继续往下走进入 while 循环，如果传进来的状态小于队列中的最大状态，`backwardPass()`将队列中所有大于当前状态的观察者同步到当前状态；如果存在队列中的状态小于当前状态的，那么通过`forwardPass()`将队列中所有小于当前状态的观察者同步到当前状态。同步过程都会执行到`ObserverWithState#dispatchEvent()`方法：



```
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```



`ObserverWithState#dispatchEvent()`中调用了`mLifecycleObserver.onStateChanged()`，这个`mLifecycleObserver`是`LifecycleEventObserver`类型 (父类是接口 LifecycleObserver
)，在构造方法中通过`Lifecycling.lifecycleEventObserver()`创建的，最终返回的是`ReflectiveGenericLifecycleObserver`：



```
//ReflectiveGenericLifecycleObserver.java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```



`ClassesInfoCache`内部存了所有观察者的回调信息，`CallbackInfo`是当前观察者的回调信息。`getInfo()`中如果从内存`mCallbackMap`中有对应回调信息，直接返回；否则通过`createInfo()`内部解析注解`OnLifecycleEvent`对应的方法并最终生成`CallbackInfo`返回。

```
//ClassesInfoCache.java
CallbackInfo getInfo(Class<?> klass) {

    CallbackInfo existing = mCallbackMap.get(klass);
    if (existing != null) {
        return existing;
    }
    existing = createInfo(klass, null);
    return existing;
}

private void verifyAndPutHandler(Map<MethodReference, Lifecycle.Event> handlers,
        MethodReference newHandler, Lifecycle.Event newEvent, Class<?> klass) {
    Lifecycle.Event event = handlers.get(newHandler);
    if (event == null) {
        handlers.put(newHandler, newEvent);
    }
}

private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
    Class<?> superclass = klass.getSuperclass();
    Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
    if (superclass != null) {
        CallbackInfo superInfo = getInfo(superclass);
        if (superInfo != null) {
            handlerToEvent.putAll(superInfo.mHandlerToEvent);
        }
    }

    Class<?>[] interfaces = klass.getInterfaces();
    for (Class<?> intrfc : interfaces) {
        for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                intrfc).mHandlerToEvent.entrySet()) {
            verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
        }
    }

    Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
    boolean hasLifecycleMethods = false;
    //遍历寻找OnLifecycleEvent注解对应的方法
    for (Method method : methods) {
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation == null) {
            continue;
        }
        hasLifecycleMethods = true;
        Class<?>[] params = method.getParameterTypes();
        int callType = CALL_TYPE_NO_ARG;
        if (params.length > 0) {
            callType = CALL_TYPE_PROVIDER;
            //第一个方法参数必须是LifecycleOwner
            if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. Must be one and instanceof LifecycleOwner");
            }
        }
        Lifecycle.Event event = annotation.value();

        if (params.length > 1) {
            callType = CALL_TYPE_PROVIDER_WITH_EVENT;
            //第2个参数必须是Lifecycle.Event
            if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. second arg must be an event");
            }
            //当有2个参数时，注解必须是Lifecycle.Event.ON_ANY
            if (event != Lifecycle.Event.ON_ANY) {
                throw new IllegalArgumentException(
                        "Second arg is supported only for ON_ANY value");
            }
        }
        if (params.length > 2) {
            throw new IllegalArgumentException("cannot have more than 2 params");
        }
        MethodReference methodReference = new MethodReference(callType, method);
        verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
    }
    CallbackInfo info = new CallbackInfo(handlerToEvent);
    mCallbackMap.put(klass, info);
    mHasLifecycleMethods.put(klass, hasLifecycleMethods);
    return info;
}

//CallbackInfo.java
static class CallbackInfo {
    final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;
    final Map<MethodReference, Lifecycle.Event> mHandlerToEvent;

    CallbackInfo(Map<MethodReference, Lifecycle.Event> handlerToEvent) {
        mHandlerToEvent = handlerToEvent;
        mEventToHandlers = new HashMap<>();
        for (Map.Entry<MethodReference, Lifecycle.Event> entry : handlerToEvent.entrySet()) {
            Lifecycle.Event event = entry.getValue();
            List<MethodReference> methodReferences = mEventToHandlers.get(event);
            if (methodReferences == null) {
                methodReferences = new ArrayList<>();
                mEventToHandlers.put(event, methodReferences);
            }
            methodReferences.add(entry.getKey());
        }
    }
    
void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
    invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
    invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
            target);
}

private static void invokeMethodsForEvent(List<MethodReference> handlers,
        LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
    if (handlers != null) {
        for (int i = handlers.size() - 1; i >= 0; i--) {
            handlers.get(i).invokeCallback(source, event, mWrapped);
        }
    }
}
```

最终调用到了`MethodReference#invokeCallback()`：

```
//MethodReference.java
static class MethodReference {
    final int mCallType;
    final Method mMethod;

    MethodReference(int callType, Method method) {
        mCallType = callType;
        mMethod = method;
        mMethod.setAccessible(true);
    }

    void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
        //noinspection TryWithIdenticalCatches
        try {
            //OnLifecycleEvent注解对应的方法入参
            switch (mCallType) {
                //没有参数
                case CALL_TYPE_NO_ARG:
                    mMethod.invoke(target);
                    break;
                //一个参数：LifecycleOwner
                case CALL_TYPE_PROVIDER:
                    mMethod.invoke(target, source);
                    break;
                //两个参数：LifecycleOwner，Event
                case CALL_TYPE_PROVIDER_WITH_EVENT:
                    mMethod.invoke(target, source, event);
                    break;
            }
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Failed to call observer method", e.getCause());
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }

        MethodReference that = (MethodReference) o;
        return mCallType == that.mCallType && mMethod.getName().equals(that.mMethod.getName());
    }

    @Override
    public int hashCode() {
        return 31 * mCallType + mMethod.getName().hashCode();
    }
}
```

根据不同入参个数通过反射来初始化并执行观察者相应方法，整个流程就从`LifecycleOwner`中的生命周期`Event`传到了`LifecycleObserver`中对应的方法。到这里整个流程就差不多结束了，最后是`LifecycleOwner`的子类`LifecycleRegistry`添加观察者的过程：

```
//LifecycleRegistry.java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    //key是LifecycleObserver,value是ObserverWithState
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
    //如果已经存在，直接返回
    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    //目标State
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    //循环遍历，将目标State连续同步到Observer中
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}

private State calculateTargetState(LifecycleObserver observer) {
    Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

    State siblingState = previous != null ? previous.getValue().mState : null;
    State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
            : null;
    return min(min(mState, siblingState), parentState);
}
```

添加观察者，并通过`while`循环，将最新的`State`状态连续同步到`Observer`中，虽然可能添加`Observer`比`LifecyleOwner`分发事件晚，但是依然能收到所有事件，类似于事件总线的粘性事件。最后画一下整体的类图关系：

![img](https://img-blog.csdnimg.cn/img_convert/3aac978ae99940553f232df89d3c3716.png)