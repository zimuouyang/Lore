## ViewModel 介绍

ViewModel 的定义：**ViewModel 旨在以注重生命周期的方式存储和管理界面相关的数据**。ViewModel 本质上是视图（View）与数据（[Model](https://so.csdn.net/so/search?q=Model&spm=1001.2101.3001.7020)）之间的桥梁，想想以前的 MVC 模式，视图和数据都会写在 Activity/Fragment 中，导致 Activity/Fragment 过重，后续难以维护，而 ViewModel 将视图和数据进行了分离解耦，为视图层提供数据。

ViewModel 的特点：

**ViewModel 生命周期比 Activity 长**

数据可在屏幕发生旋转等配置更改后继续留存。下面是 ViewModel [生命周期](https://so.csdn.net/so/search?q=生命周期&spm=1001.2101.3001.7020)图：

![img](https://img-blog.csdnimg.cn/img_convert/ca45e7374bd273104ea2ba69fc6da2ce.png)

可以看到即使是发生屏幕旋转，旋转之后拿到的 ViewModel 跟之前的是同一个实例，即发生屏幕旋转时，ViewModel 并不会消失重建；而如果 Activity 是正常 finish()，ViewModel 则会调用 onClear() 销毁。

**ViewModel 中不能持有 Activity 引用**

不要将 Activity 传入 ViewModel 中，因为 ViewModel 的生命周期比 Activity 长，所以如果 ViewModel 持有了 Activity 引用，很容易造成内存泄漏。如果想在 ViewModel 中使用 Application，可以使用 ViewModel 的子类 AndroidViewModel，其在构造方法中需要传入了 Application 实例：

```JAVA
public class AndroidViewModel extends ViewModel {
    private Application mApplication;

    public AndroidViewModel(@NonNull Application application) {
        mApplication = application;
    }

    public <T extends Application> T getApplication() {
        return (T) mApplication;
    }
}
```

## ViewModel 使用举例

引入 ViewModel，在介绍 [Jetpack](https://so.csdn.net/so/search?q=Jetpack&spm=1001.2101.3001.7020) 系列文章 Lifecycle 时已经提过一次，这里再写一下：

```java
implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0"
```

页面中有两个 [Fragment](https://so.csdn.net/so/search?q=Fragment&spm=1001.2101.3001.7020)，左侧为列表，右侧为详情，当点击左侧某一个 Item 时，右侧会展示相应的数据，即两个 Fragment 可以通过 ViewModel 进行通信；同时可以看到，当屏幕发生旋转的时候，右侧详情页的数据并没有丢失，而是直接进行了展示。

```java
//ViewModelActivity.kt
class ViewModelActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Log.e(JConsts.VIEW_MODEL, "onCreate")
        setContentView(R.layout.activity_view_model)
    }
}
```

其中的 XML 文件:

```java
//activity_view_model.xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".viewmodel.ViewModelActivity">

    <fragment
        android:id="@+id/fragment_item"
        android:name="com.example.jetpackstudy.viewmodel.ItemFragment"
        android:layout_width="150dp"
        android:layout_height="match_parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toLeftOf="@+id/fragment_detail"
        app:layout_constraintTop_toTopOf="parent" />

    <fragment
        android:id="@+id/fragment_detail"
        android:name="com.example.jetpackstudy.viewmodel.DetailFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        app:layout_constraintLeft_toRightOf="@+id/fragment_item"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

直接将 Fragment 以布局方式写入我们的 Activity 中，继续看两个 Fragment：

```java
//左侧列表Fragment
class ItemFragment : Fragment() {
    lateinit var mRvView: RecyclerView

    //Fragment之间通过传入同一个Activity来共享ViewModel
    private val mShareModel by lazy {
        ViewModelProvider(activity!!).get(ShareViewModel::class.java)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val view = LayoutInflater.from(activity)
            .inflate(R.layout.layout_fragment_item, container, false)
        mRvView = view.findViewById(R.id.rv_view)
        mRvView.layoutManager = LinearLayoutManager(activity)
        mRvView.adapter = MyAdapter(mShareModel)

        return view
    }

    //构造RecyclerView的Adapter
    class MyAdapter(private val sViewModel: ShareViewModel) :
        RecyclerView.Adapter<MyAdapter.MyViewHolder>() {

        class MyViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
            val mTvText: TextView = itemView.findViewById(R.id.tv_text)
        }

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
            val itemView = LayoutInflater.from(parent.context)
                .inflate(R.layout.item_fragment_left, parent, false)
            return MyViewHolder(itemView)
        }

        override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
            val itemStr = "item pos:$position"
            holder.mTvText.text = itemStr
            //点击发送数据
            holder.itemView.setOnClickListener {
                sViewModel.clickItem(itemStr)
            }
        }

        override fun getItemCount(): Int {
            return 50
        }
    }
}
```



```java
//右侧详情页Fragment
class DetailFragment : Fragment() {
    lateinit var mTvDetail: TextView

    //Fragment之间通过传入同一个Activity来共享ViewModel
    private val mShareModel by lazy {
        ViewModelProvider(activity!!).get(ShareViewModel::class.java)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return LayoutInflater.from(context)
            .inflate(R.layout.layout_fragment_detail, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        mTvDetail = view.findViewById(R.id.tv_detail)
        //注册Observer并监听数据变化
        mShareModel.itemLiveData.observe(activity!!, { itemStr ->
            mTvDetail.text = itemStr
        })
    }
}
```

最后贴一下我们的 ViewModel:

```java
//ShareViewModel.kt
class ShareViewModel : ViewModel() {
    val itemLiveData: MutableLiveData<String> by lazy { MutableLiveData<String>() }

    //点击左侧Fragment中的Item发送数据
    fun clickItem(infoStr: String) {
        itemLiveData.value = infoStr
    }
}
```

这里再强调一下，两个 Fragment 中 ViewModelProvider(activity).get() 传入的是同一个 Activity，从而得到的 ViewModel 是同一个实例，进而可以进行互相通信。

上面使用 ViewModel 的写法有两个好处：

**屏幕切换时保存数据**

- 屏幕发生变化时，不需要重新请求数据，直接从 ViewModel 中再次拿数据即可。

**Fragment 之间共享数据**：

- Activity 不需要执行任何操作，也不需要对此通信有任何了解。
- 除了 SharedViewModel 约定之外，Fragment 不需要相互了解。如果其中一个 Fragment 消失，另一个 Fragment 将继续照常工作。
- 每个 Fragment 都有自己的生命周期，而不受另一个 Fragment 的生命周期的影响。如果一个 Fragment 替换另一个 Fragment，界面将继续工作而没有任何问题。

## ViewModel 与 onSaveInstance(Bundle) 对比

- ViewModel 是将数据存到内存中，而 onSaveInstance() 是通过 Bundle 将序列化数据存在磁盘中
- ViewModel 可以存储任何形式的数据，且大小不限制 (不超过 App 分配的内存即可)，onSaveInstance() 中只能存储可序列化的数据，且大小一般不超过 1M（IPC 通信数据限制）

## [源码](https://so.csdn.net/so/search?q=源码&spm=1001.2101.3001.7020)解析

### ViewModel 的存取

我们在获取 ViewModel 实例时，并不是直接 new 出来的，而是通过 ViewModelProvider.get() 获取的，顾名思义，ViewModelProvider 意为 ViewModel 提供者，那么先来看它的构造方法：

```java
//ViewModelProvider.java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}

public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
    this(owner.getViewModelStore(), factory);
}

public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}
```

ViewModelProvider 构造函数中的几个参数：

- ViewModelStoreOwner：ViewModel 存储器拥有者，用来提供 ViewModelStore
- ViewModelStore：ViewModel 存储器，用来存储 ViewModel
- Factory：创建 ViewModel 的工厂

```java
private final ViewModelStore mViewModelStore;

private static final String DEFAULT_KEY =
        "androidx.lifecycle.ViewModelProvider.DefaultKey";

@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    //首先构造了一个key，直接调用下面的get(key,modelClass)
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    //尝试从ViewModelStore中获取ViewModel
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        //viewModel不为空直接返回该实例
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
    } else {
        //viewModel为空，通过Factory创建
        viewModel = mFactory.create(modelClass);
    }
    //将ViewModel保存到ViewModelStore中
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```

逻辑很简单，首先尝试通过 ViewModelStore.get(key) 获取 ViewModel，如果不为空直接返回该实例；如果为空，通过 Factory.create 创建 ViewModel 并保存到 ViewModelStore 中。先来看 Factory 是如何创建 ViewModel 的，ViewModelProvider 构造函数中，如果没有传入 Factory，那么会使用 NewInstanceFactory：

```java
//接口Factory
public interface Factory {
   <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}

//默认Factory的实现类NewInstanceFactory
public static class NewInstanceFactory implements Factory {
   private static NewInstanceFactory sInstance;

   //获取单例
   static NewInstanceFactory getInstance() {
       if (sInstance == null) {
          sInstance = new NewInstanceFactory();
         }
        return sInstance;
       }

      @Override
   public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
       try {
           //反射创建
           return modelClass.newInstance();
       } catch (InstantiationException e) {
           throw new RuntimeException("Cannot create an instance of " + modelClass, e);
       } catch (IllegalAccessException e) {
           throw new RuntimeException("Cannot create an instance of " + modelClass, e);
       }
   }
}
```

可以看到 NewInstanceFactory 的实现很简单，直接通过传入的 class 反射创建实例，泛型中限制必须是 ViewModel 的子类，所以最终创建的是 ViewModel 实例。

看完 Factory，接着来看 ViewModelStore：

```java
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        //如果oldViewModel不为空，调用oldViewModel的onCleared释放资源
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    //遍历存储的ViewModel并调用其clear()方法，然后清除所有ViewModel
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```



ViewModelStore 类也很简单，内部就是通过 Map 进行存储 ViewModel 的。到这里，我们基本看完了 ViewModel 的存取，流程如下：



![img](https://img-blog.csdnimg.cn/img_convert/61bfe8240cd5d1648037da8bdc77ea8e.png)

### ViewModelStore 的存取

上面聊了 ViewModel 的存取，有一个重要的点没有说到，既然 ViewModel 的生命周期比 Activity 长，而 ViewModel 又是通过 ViewModelStore 存取的，那么 ViewModelStore 又是如何存取的呢？在上面的流程图中我们知道 ViewModelStore 是通过 ViewModelStoreOwner 提供的：

```java
//接口ViewModelStoreOwner.java
public interface ViewModelStoreOwner {
    ViewModelStore getViewModelStore();
}
```

ViewModelStoreOwner 中接口方法 getViewModelStore() 返回的既是 ViewModelStore。我们上面例子获取 ViewModel 时是`ViewModelProvider(activity).get(ShareViewModel::class.java)`，其中的 activity 其实就是 ViewModelStoreOwner，也就是 Activity 中实现了这个接口：

```java
//ComponentActivity.java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,.... {

    @Override
    public ViewModelStore getViewModelStore() {
      //Activity还未关联到Application时，会抛异常，此时不能使用ViewModel
      if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        ensureViewModelStore();
        return mViewModelStore;
    }

    void ensureViewModelStore() {
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                //从NonConfigurationInstances中恢复ViewModelStore
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
    }

    @Override
    //覆写了Activity的onRetainNonConfigurationInstance方法，在Activity#retainNonConfigurationInstances()方法中被调用，即配置发生变化时调用。
    public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
        //尝试从之前的存储中获取NonConfigurationInstance
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
          if (nc != null) {
              //从NonConfigurationInstances中恢复ViewModelStore
              viewModelStore = nc.viewModelStore;
          }
      }

     if (viewModelStore == null && custom == null) {
         return null;
     }
     //如果viewModelStore不为空，当配置变化时将ViewModelStore保存到NonConfigurationInstances中
     NonConfigurationInstances nci = new NonConfigurationInstances();
     nci.custom = custom;
     nci.viewModelStore = viewModelStore;
     return nci;
 }

    //内部类NonConfigurationInstances
    static final class NonConfigurationInstances {
       Object custom;
       ViewModelStore viewModelStore;
   }

}
```

NonConfigurationInstances 是 ComponentActivity 的静态内部类，里面包含了 ViewModelStore。getViewModelStore() 内部首先尝试通过 getLastNonConfigurationInstance() 来获取 NonConfigurationInstances，不为空直接能拿到对应的 ViewModelStore；否则直接 new 一个新的 ViewModelStore

跟进去 getLastNonConfigurationInstance() 这个方法：

```java
//Activity.java
public class Activity extends ContextThemeWrapper {

  NonConfigurationInstances mLastNonConfigurationInstances;

  public Object getLastNonConfigurationInstance() {
      return mLastNonConfigurationInstances != null
            ? mLastNonConfigurationInstances.activity : null;
  }

  NonConfigurationInstances retainNonConfigurationInstances() {
     //onRetainNonConfigurationInstance实现在子类ComponentActivity中实现
     Object activity = onRetainNonConfigurationInstance();
     .......

     if (activity == null && children == null && fragments == null && loaders == null
            && mVoiceInteractor == null) {
         return null;
     }
     NonConfigurationInstances nci = new NonConfigurationInstances();
     nci.activity = activity;
     ......
     return nci;
 }

  final void attach(Context context,
        .......,
        NonConfigurationInstances lastNonConfigurationInstances) {
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
   }

  static final class NonConfigurationInstances {
      Object activity;
      ......
  }
}
```



可以看到 getLastNonConfigurationInstance() 中是通过 mLastNonConfigurationInstances 是否为空来判断的，搜索一下该变量赋值的地方，就找到了 Activity#attach() 方法。我们知道 Activity#attach() 是在创建 Activity 的时候调用的，顺藤摸瓜就可以找到了 ActivityThread：

```java
//ActivityThread.java
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();

Activity.NonConfigurationInstances lastNonConfigurationInstances;

//1、将NonConfigurationInstances存储到ActivityClientRecord
ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
        int configChanges, boolean getNonConfigInstance, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    Class<? extends Activity> activityClass = null;
    if (r != null) {
        ......
        if (getNonConfigInstance) {
            try {
                //retainNonConfigurationInstances
                r.lastNonConfigurationInstances
                        = r.activity.retainNonConfigurationInstances();
            } catch (Exception e) {
                ......
            }
        }
    }
    synchronized (mResourcesManager) {
        mActivities.remove(token);
    }
    return r;
}

//2、从ActivityClientRecord中获取NonConfigurationInstances
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  ......
  activity.attach(...., r.lastNonConfigurationInstances,....);
}
```

- 在执行 performDestroyActivity() 的时候，会调用 Activity#retainNonConfigurationInstances() 方法将生成的 NonConfigurationInstances 赋值给 lastNonConfigurationInstances。
- 在 performLaunchActivity() 中又会通过 Activity#attach() 将 lastNonConfigurationInstances 赋值给 Activity.mLastNonConfigurationInstances，进而取到 ViewModelStore。

`ViewModelStore`的存取都是间接在`ActivityThread`中进行并保存在`ActivityClientRecord`中。在`Activity`配置变化时，`ViewModelStore`可以在`Activity`销毁时得以保存并在重建时重新从`lastNonConfigurationInstances`中获取，又因为`ViewModelStore`提供了`ViewModel`，所以`ViewModel`也可以在`Activity`配置变化时得以保存，这也是为什么`ViewModel`的生命周期比`Activity`生命周期长的原因了。