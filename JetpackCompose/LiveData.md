## LiveData 介绍

**LiveData 是一种可观察的数据存储类**。LiveData 具有生命周期感知能力，遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的 Observer，非活跃状态下的 Observer 不会受到通知。

生命周期状态可以通过 Lifecycle 提供，包括 DESTROYED、INITIALIZED、CREATED、STARTED、RESUMED，当且仅当生命周期处于 STARTED、RESUMED 时为活跃状态，其他状态是非活跃状态。

## LiveData 优点

- 确保界面符合数据状态
  LiveData 遵循观察者模式。当数据发生变化时，LiveData 会通知 Observer 对象，那么 Observer 回调的方法中就可以进行 UI 更新，即数据驱动。
- 不会发生内存泄漏
  观察者会绑定到 Lifecycle 对象，并在其关联的生命周期遭到销毁（如 Activity 进入 ONDESTROY 状态）后进行自我清理。
- 不会因 Activity 停止而导致崩溃
  如果观察者的生命周期处于非活跃状态（如返回栈中的 Activity），则它不会接收任何 LiveData 事件。
- 不再需要手动处理生命周期
  界面组件只是观察相关数据，不会停止或恢复观察。LiveData 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。
- 数据始终保持最新状态
  如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 Activity 会在返回前台后立即接收最新的数据。
- 配置更改时自动保存数据
  如果由于配置更改（如设备旋转）而重新创建了 Activity 或 Fragment，它会立即接收最新的可用数据。
- 共享资源
  使用单例模式扩展 LiveData 对象以封装系统服务，以便在应用中共享它们。LiveData 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 LiveData 对象。

## LiveData 使用举例

### 基础用法

先上效果图：

![img](https://img-blog.csdnimg.cn/img_convert/64d52447e84bba08f204d7e4a10bcaae.gif)

在 Activity 中动态添加了一个 Fragment，点击按钮产生一个 1000 以内的随机值，并通过 LiveData.setValue 发送出去，在 Fragment 中通过 LiveData.observe 进行数据观察与接收，可以看到即使 Activity 中先发送的数据，Fragment 中滞后注册观察者依然能收到数据，即 LiveData 发送的是粘性事件。

首先需要保证 Activity 和 Fragment 中的 LiveData 是同一个实例：

```java
//LiveDataInstance.kt 使用object来声明单例模式
object LiveDataInstance {
    //MutableLiveData是抽象类LiveData的具体实现类
    val INSTANCE = MutableLiveData<String>()
}
```

Activity 中随机生成一个数并通过 LiveData.setValue 进行发送：

```java
//LiveDataActivity.kt
class LiveDataActivity : AppCompatActivity() {
    lateinit var mTextView: TextView
    var mFragment: LiveDataFragment? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)
        mTextView = findViewById(R.id.tv_text)
        addLiveDataFragment()
    }

    fun updateValue(view: View) {
        sendData(produceData())
    }

    //随机更新一个整数
    private fun produceData(): String {
        val randomValue = (0..1000).random().toString()
        mTextView.text = "Activity中发送：$randomValue"
        return randomValue
    }

    //通过setValue发送更新
    private fun sendData(randomValue: String) {
        LiveDataInstance.INSTANCE.value = randomValue
    }

    //添加Fragment
    fun addFragment(view: View) {
        addLiveDataFragment()
    }

    //移除Fragment
    fun removeFragment(view: View) {
        delLiveDataFragment()
    }

    private fun addLiveDataFragment() {
        val fragment = supportFragmentManager.findFragmentById(R.id.fl_content)
        if (fragment != null) {
            Toast.makeText(this, "请勿重复添加", Toast.LENGTH_SHORT).show()
            return
        }

        if (mFragment == null) {
            mFragment = LiveDataFragment.newInstance()
        }
        supportFragmentManager
            .beginTransaction()
            .add(R.id.fl_content, mFragment!!)
            .commitAllowingStateLoss()
    }

    private fun delLiveDataFragment() {
        val fragment = supportFragmentManager.findFragmentById(R.id.fl_content)
        if (fragment == null) {
            Toast.makeText(this, "没有Fragment", Toast.LENGTH_SHORT).show()
            return
        }
        supportFragmentManager.beginTransaction().remove(fragment).commitAllowingStateLoss()
    }
}
```

Fragment 动态添加到 Activity 中，并通过 LiveData.observe 注册观察者并监听数据变化：

```java
//LiveDataFragment.kt
class LiveDataFragment : Fragment() {
    lateinit var mTvObserveView: TextView

    //数据观察者 数据改变时在onChange()中进行刷新
    private val changeObserver = Observer<String> { value ->
        value?.let {
            Log.e(JConsts.LIVE_DATA, "observer:$value")
            mTvObserveView.text = value
        }
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.live_data_fragment, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        mTvObserveView = view.findViewById(R.id.tv_observe)
    }

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        //通过LiveData.observe注册观察者，监听数据变化
        LiveDataInstance.INSTANCE.observe(this, changeObserver)
    }

    companion object {
        fun newInstance() = LiveDataFragment()
    }
}
```

上面就是一个 LiveData 的基本用法了，很简单，Activity/Fragment 中共用 LiveData 实例，在 Activity 中通过点击按钮生成一个随机数并通过 LiveData.setValue 发送数据 (如果在子线程中发送，需要使用 postValue)，然后 Fragment 中通过 LiveData.observe 注册观察者并监听数据变化。

改一下代码：

```java
 //LiveDataActivity.kt
 override fun onStop() {
     super.onStop()
     val data = produceData()
     Log.e(JConsts.LIVE_DATA, "onStop():$data")
     sendData(data)
 }

 //LiveDataFragment.kt
 private val changeObserver = Observer<String> { value ->
     value?.let {
         Log.e(JConsts.LIVE_DATA, "observer:$value")
         mTvObserveView.text = value
     }
 }
```

点击 Home 键，Activity 的 onStop 会触发，并通过 LiveData.setValue 发送数据，看下打印日志：

```
2021-07-13 17:07:08.662 1459-1459/com.example.jetpackstudy E/LIVEDATA: onStop():742
```

Activity 中在 onStop 中重新生成了一个随机值并发送了出去，但是在 Fragment 中的 Observer 中并没有收到数据，这是为什么呢？还记得 LiveData 的能力吗，它是能感知生命周期的，并且只会更新处于活跃状态下的 Observer（STARTED、RESUMED 状态），所以在 onStop 中发送的事件，Fragment 作为观察者已经不在活跃状态下了，并不会收到通知，当我们 App 切回前台时，Observer 重新回到活跃状态，所以会收到 Activity 之前发送的事件：

```
2021-07-13 17:12:47.433 5850-5850/com.example.jetpackstudy E/LIVEDATA: observer:742
```

如果想让 Observer 不管在什么状态下都能马上收到数据变化的通知，可以使用 LiveData.observeForever 来注册并监听数据变化：

```java
 //LiveDataFragment.kt
 private val changeObserver = Observer<String> { value ->
     value?.let {
         Log.e(JConsts.LIVE_DATA, "observer:$value")
         mTvObserveView.text = value
     }
 }

//observeForever不管Observer是否处于活跃状态都会立马相应数据变化
//注意这里只需要传一个Observer即可，不需要传入LifecycleOwner，因为不需要考虑Observer是否处于活跃状态
LiveDataInstance.INSTANCE.observeForever(changeObserver)

override fun onDestroy() {
    super.onDestroy()
    //需要手动移除观察者
    LiveDataInstance.INSTANCE.observeForever(changeObserver)
}
```

上述代码重新实验，点击 Home 键将 App 切到后台：

```
2021-07-13 17:29:56.848 15679-15679/com.example.jetpackstudy E/LIVEDATA: onStop():878
2021-07-13 17:29:56.849 15679-15679/com.example.jetpackstudy E/LIVEDATA: observer:878
```

可以看到通过 LiveData.observeForever 注册的 Observer 即使不在活跃状态也是会立马相应数据变化的，这里要注意一点，LiveData.observeForever 注册的 Observer 并不会自动解除注册，需要我们手动处理。

### 进阶用法

#### Transformations.map() 修改数据源

先上效果图：

![img](https://img-blog.csdnimg.cn/img_convert/a3476b15a0800d013f80b79d7b062a99.gif)

```java
//LiveDataFragment.kt
//数据观察者 数据改变时在onChange()中进行刷新
private val changeObserver = Observer<String> { value ->
    value?.let {
        Log.e(JConsts.LIVE_DATA, "transform observer:$value")
        mTvObserveView.text = value
    }
}

//Transformations.map()改变接收的data
val transformLiveData = Transformations.map(LiveDataInstance.INSTANCE) { "Transform:$it" }

//观察者监听的时候传入了LifecycleOwner 用以监听生命周期变化
transformLiveData.observe(this, changeObserver)
```

可以看到在 Activity 中发送的数据源是 “xxx”，Fragment 中经过 Transformations.map 变换后变成 "Transform:xxx"，通过 Transformations.map() 可以对接收的数据源进行修改。

#### Transformations.switchMap() 切换数据源

![img](https://img-blog.csdnimg.cn/img_convert/660d294317ecf4514129d6b3dbda15c4.gif)

```java
//LiveDataInstance.kt
object LiveDataInstance {
    val INSTANCE = MutableLiveData<String>()
    val INSTANCE2 = MutableLiveData<String>()
    val SWITCH = MutableLiveData<Boolean>()
}
```

**注：一般 LiveData 都是与 ViewModel 结合使用的，本文主要介绍 LiveData，所以使用了单例**

```java
//LiveDataActivity.kt
mBtnSwitch = findViewById(R.id.btn_switch)
mBtnSwitch.setOnCheckedChangeListener { _, isChecked ->
//发送开关状态 用以在Transformations.switchMap中切换数据源
LiveDataInstance.SWITCH.value = isChecked
}

//通过setValue发送更新
private fun sendData(randomValue: String, isLeft: Boolean) {
  if (isLeft) {
    LiveDataInstance.INSTANCE.value = randomValue
  } else {
    LiveDataInstance.INSTANCE2.value = randomValue
  }
}
```

```java
//LiveDataFragment.kt
//数据观察者 数据改变时在onChange()中进行刷新
private val changeObserver = Observer<String> { value ->
    value?.let {
        Log.e(JConsts.LIVE_DATA, "transform observer:$value")
        mTvObserveView.text = value
    }
}

//Transformations.switchMap()切换数据源
val switchMapLiveData =
   Transformations.switchMap(LiveDataInstance.SWITCH) { switchRight ->
       if (switchRight) {
            LiveDataInstance.INSTANCE2
        } else {
            LiveDataInstance.INSTANCE
        }
     }
switchMapLiveData.observe(this, changeObserverTransform)
```

例子中有两个数据源：LiveDataInstance.INSTANCE、LiveDataInstance.INSTANCE2，当 Switch 开关切换时，通过 Transformations.switchMap() 可以来回切换数据源，Observer 中也会更新对应的数据。

## 源码解析

### 发送数据 setValue/postValue

```
//LiveData.java
    //setValue发送数据，只能在主线程中使用
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }

    //postValue发送数据，可以在子线程中使用
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
 ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }

private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            setValue((T) newValue);
        }
    };
```

可以看到 setValue/postValue 都可以发送数据，区别是 postValue 还可以在子线程中发送数据，本质上 postValue 通过 Handler 将事件发送到 Main 线程中，最终也是调用了 setValue 发送事件，所以只看 setValue() 方法，该方法最后调用了 dispatchingValue() 方法并传入了一个参数 null，继续看该方法：

```
    void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                //2、通过observe()的方式会调用这里
                considerNotify(initiator);
                initiator = null;
            } else {
                //1、通过setValue/postValue的方式会调用这里，遍历所有观察者并进行分发
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }

    private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            //观察者不在活跃状态 直接返回
            return;
        }
        //如果是observe()，则是在STARTED、RESUMED状态时活跃；如果是ObserveForever()，则认为一直是活跃状态
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        //Observer中的Version必须小于LiveData中的Version，防止重复发送
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        //回调Observer的onChange方法并接收数据
        observer.mObserver.onChanged((T) mData);
    }
```

因为传入的参数是 null，所以最终走到了 1 处，遍历所有的观察者并回调 Observer 的 onChange 方法接收数据，这样就完成了一次数据的传递。2 处是单独调用一个观察者并回调其 onChange 方法接收数据，是执行 observe() 方法的时候执行的，具体等后面分析。

### 注册观察者 Observer 并监听数据变化

#### LiveData.observe()

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        //如果当前观察者处于DESTROYED状态，直接返回
        return;
    }
    //将LifecycleOwner、Observer包装成LifecycleBoundObserver
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    //ObserverWrapper是LifecycleBoundObserver的父类
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    //如果mObservers中存在该Observer且跟传进来的LifecycleOwner不同，直接抛异常，一个Observer只能对应一个LifecycleOwner
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    //如果已经存在Observer且跟传进来的LifecycleOwner是同一个，直接返回
    if (existing != null) {
        return;
    }
    //通过Lifecycle添加观察者
    owner.getLifecycle().addObserver(wrapper);
}
```

可以看到最后 observe() 将 Observer 加入到 Lifecycle 里去了，并通过 onStateChanged() 回调来监听 LifecycleOwner 生命周期的变化，主要看 onStateChanged() 方法：

```java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        //Observer对应的LifecycleOwner是DESTROYED状态，直接删除该Observer，所以LiveData有自动解除Observer的能力
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        //
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}

//ObserverWrapper.java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    if (wasInactive && mActive) {
        //观察者数量从0变为1时
        onActive();
    }
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        //观察者数量从1变为0时
        onInactive();
    }
    if (mActive) {
        //观察者为活跃状态，进行分发
        dispatchingValue(this);
    }
}
```

onActive() 在观察者数量从 0 变为 1 时执行；onInactive() 在观察者数量从 1 变为 0 时执行。最后如果当前观察者是活跃状态，直接执行 dispatchingValue(this)，this 是当前 ObserverWrapper 对象，还记得 dispatchingValue() 方法吗，前面讲这个方法的时候留了个疑问，这里就会执行前面讲的该方法里 2 处的代码，用以分发事件并在 Observer 的 onChange() 方法里接收并处理事件。

#### LiveData.observeForever()

```
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    assertMainThread("observeForever");
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```

observeForever() 中不需要传 LifecycleOwner 参数，因为 observeForever() 认为是一直活跃的状态，所以不需要监听 LifecycleOwner 的生命周期，最后是直接执行了 wrapper.activeStateChanged(true) 方法，后续的逻辑跟上面 observe() 一样了。**这里注意一点，observeForever() 注册的观察者当处于 DESTROYED 的时候并不会自动删除，需要手动删除之**。

### LiveData 实现类 MutableLiveData

```
public class MutableLiveData<T> extends LiveData<T> {

    public MutableLiveData(T value) {
        super(value);
    }

    public MutableLiveData() {
        super();
    }

    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

MutableLiveData 是抽象类 LiveData 的具体实现类。

### 数据切换 / 修改 Transformations.map()/switchMap()

```java
//Transformations.java
public static <X, Y> LiveData<Y> map(
        @NonNull LiveData<X> source,
        @NonNull final Function<X, Y> mapFunction) {
    final MediatorLiveData<Y> result = new MediatorLiveData<>();
    result.addSource(source, new Observer<X>() {
        @Override
        public void onChanged(@Nullable X x) {
            result.setValue(mapFunction.apply(x));
        }
    });
    return result;
}
```

从源码的注释上，看到了这么一句话`This method is analogous to {@link io.reactivex.Observable#map}`，哦，原来用法是跟 RxJava 中的 Map 操作符类似。第一个参数是源 LiveData<X>，第 2 个参数是个 Funtion< X,Y>，目的就是将 LiveData< X > 变换为 LiveData< Y>，然后再重新发送事件。map() 方法里 new 了一个 MediatorLiveData 并执行了 addSource() 方法，看看这个方法怎么实现的：

```
//MediatorLiveData.java
public class MediatorLiveData<T> extends MutableLiveData<T> {
    private SafeIterableMap<LiveData<?>, Source<?>> mSources = new SafeIterableMap<>();

    @MainThread
    public <S> void addSource(@NonNull LiveData<S> source, @NonNull Observer<? super S> onChanged) {
        //将源LiveData及Observer包装成Source
        Source<S> e = new Source<>(source, onChanged);
        Source<?> existing = mSources.putIfAbsent(source, e);
        //如果源LiveData中已经有Observer且跟传进来的不一致，直接抛异常
        if (existing != null && existing.mObserver != onChanged) {
            throw new IllegalArgumentException(
                    "This source was already added with the different observer");
        }
        if (existing != null) {
            return;
        }
        if (hasActiveObservers()) {
            //判断有活跃观察者时
            e.plug();
        }
    }

    @MainThread
    public <S> void removeSource(@NonNull LiveData<S> toRemote) {
        Source<?> source = mSources.remove(toRemote);
        if (source != null) {
            source.unplug();
        }
    }

    private static class Source<V> implements Observer<V> {
        final LiveData<V> mLiveData;
        final Observer<? super V> mObserver;
        int mVersion = START_VERSION;

        Source(LiveData<V> liveData, final Observer<? super V> observer) {
            mLiveData = liveData;
            mObserver = observer;
        }

        void plug() {
            //通过observeForever添加观察者，有变动时就会回调下面的onChange()方法
            mLiveData.observeForever(this);
        }

        void unplug() {
            mLiveData.removeObserver(this);
        }

        @Override
        public void onChanged(@Nullable V v) {
            if (mVersion != mLiveData.getVersion()) {
                mVersion = mLiveData.getVersion();
                mObserver.onChanged(v);
            }
        }
    }
}
```

首先将源 LiveData 及 Observer 包装成 Source，经过了几次判断，最后执行到了 Source#plug() 方法，里面通过 observeForever 添加观察者，有变动时就会回调 Source#onChange() 方法，而这个方法里又会回调传进来的 Observer#onChange() 方法，即执行到了 map() 中传入的 Observer 的 onChange() 方法，里面通过 setValue 发送了转换之后的数据格式，这样就完成了整个的数据转换格式。那么再看 switchMap() 就简单了：

```java
public static <X, Y> LiveData<Y> switchMap(
        @NonNull LiveData<X> source,
        @NonNull final Function<X, LiveData<Y>> switchMapFunction) {
    final MediatorLiveData<Y> result = new MediatorLiveData<>();
    result.addSource(source, new Observer<X>() {
        LiveData<Y> mSource;

        @Override
        public void onChanged(@Nullable X x) {
            LiveData<Y> newLiveData = switchMapFunction.apply(x);
            if (mSource == newLiveData) {
                return;
            }
            if (mSource != null) {
                result.removeSource(mSource);
            }
            mSource = newLiveData;
            if (mSource != null) {
                result.addSource(mSource, new Observer<Y>() {
                    @Override
                    public void onChanged(@Nullable Y y) {
                        result.setValue(y);
                    }
                });
            }
        }
    });
    return result;
}
```

可以看到 switchMap() 中实现方式跟 map() 基本一致，只不过 map() 改变的是数据，而 switchMap() 改变的是数据源，可以对数据源进行切换。Transformations 还有个 distinctUntilChanged 方法简单看一下：

```java
public static <X> LiveData<X> distinctUntilChanged(@NonNull LiveData<X> source) {
    final MediatorLiveData<X> outputLiveData = new MediatorLiveData<>();
    outputLiveData.addSource(source, new Observer<X>() {

        boolean mFirstTime = true;

        @Override
        public void onChanged(X currentValue) {
            final X previousValue = outputLiveData.getValue();
            if (mFirstTime
                    || (previousValue == null && currentValue != null)
                    || (previousValue != null && !previousValue.equals(currentValue))) {
                mFirstTime = false;
                outputLiveData.setValue(currentValue);
            }
        }
    });
    return outputLiveData;
}
```

也很简单，只有当数据源发生改变时，Observer 才会相应，即发送重复的数据时，除第一次之外的数据都会被忽略。

最后画一下类图：

![img](https://img-blog.csdnimg.cn/img_convert/f0c5dd0a712bc908080eb66b5b48541f.png)