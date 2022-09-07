Jetpack 学习之 ---Hilt - 简书

## 一、介绍

Hilt 提供了一种将 Dagger 依赖注入合并到 Android 应用程序中的标准方法。

说白了，他就是针对 android 平台对 Dagger 进行了封装与扩展，并提供了 android 平台特有的一些 Component ,Scope 等, 内置了预定义绑定、预定义限定符等。

支持 Android 的类有：

Activity、Application、Fragment、View、Service、BroadcastReceiver

**[官方学习文档](https://links.jianshu.com/go?to=https%3A%2F%2Fdagger.dev%2Fhilt%2Fgradle-setup)**

## 二、基本使用

### 1、核心注解说明

#### 1) @HiltAndroidApp 注解

**所有使用 Hilt 的应用程序都必须要包含一个使用 @HiltAndroidApp 注解的 appliaction。**

因为在生成代码时，需要访问所有的 module ，编译 Application 类还需要将所有 Dagger 模块包含在其传递的依赖项中。其他组件可以访问它提供的所有的依赖项。

@HiltAndroidApp 注解是用来生成 Hilt Application 的，如果使用了 Hilt 的 Gradle Plugin，那么他的参数 value 就可以不指明，默认就是当前类的父类；如果没有使用插件，那么就需要指明 value 的类型。

#### 2) @AndroidEntryPoint 注解

该注解是表明你的依赖注入注入的位置。该注解目前仅支持在 Activity、Fragment、View、Service、BroadcastReceiver 中。

ViewModel 的注入使用 @HiltViewModel.

如果对 Fragment 使用依赖注入点，那么他所在的 Activity 也必须添加该注解。

注意：

1. hilt 仅支持扩展 ComponentActivity 的 Activity;
2. Hilt 只支持 AndroidX 下的 Fragment; 不支持保留的 Fragment;

#### 3) @InstallIn 注解

该注解表明了当前的 Module 安装在哪个 Component 上，在使用 Dagger 时，在每一个 Module 上需要加上 @Module 注解，而在 Hilt 中是使用 @Module 和 @InstallIn 注解；这其实就是 Dagger 中的一个标准的 Module。

使用了 @InstallIn 注解的 modules ，当 Component 生成后，就会被安装到关联的 Component 或 SubComponent 上去；

通过给 @InstallIn 传递参数，来指定该模块安装到哪一个 Component 中去；

#### 4) @Singleton 注解

Hilt 中的 @Singleton 注解必须要和 ApplicationComponent 组件一起使用，不能单独使用。

#### 5) @Binds 注入接口实例

将接口与实现类绑定。例如：当构造方法需要传递一个接口类型的实例对象时，就需要使用 Binds 注解来注入。

@Binds 注解使用的类必须是 abstract 类

@Binds 注解的方法必须是 abstract 抽象方法，其参数必须是接口的实例对象，返回值必须是接口类

#### 6) @Provides 注入实例

在 module 中提供的实例，不能直接通过构造函数来提供时（例如使用第三方的构建模式创建等），这时候就可以 @Providers 注解来注释一个方法专门提个这个对象实例来完成注入。

该方法的返回值必须是需要的对象的实例类型；

#### 7) @Qualifier 限定符

当你想要 Inject 的两个对象实例的类型是一样的，但是他们又是不同的实例，就可以使用 Qualifier 限定两个自定义的注解来完成.

1. 自定义 Qualifier 的注解
2. 在 module 的提供实例的方式上使用自定义的 Qualifier 注解
3. 在 Inject 的地方使用 Qualifier 注解来进行区分；

```
@Qualifier
@kotlin.annotation.Retention(AnnotationRetention.BINARY)
annotation class Binder

@Qualifier
@kotlin.annotation.Retention(AnnotationRetention.BINARY)
annotation class Customer
```

个人理解，和 Dagger 中的 Named 注解类似。

### 2、Hilt 组件的生命周期

Hilt 内置的 android 组件的生命周期如下表格所示，可以看出这些 Component 组件是随着对应的 Android 类的生命周期的创建和销毁而变化，所以我们不需要手动去管理这些组件的生命周期。

| 生成的组件                | 创建时机               | 销毁时机                |
| ------------------------- | ---------------------- | ----------------------- |
| ApplicationComponent      | Application#onCreate() | Application#onDestroy() |
| ActivityRetainedComponent | Activity#onCreate()    | Activity#onDestroy()    |
| ActivityComponent         | Activity#onCreate()    | Activity#onDestroy()    |
| FragmentComponent         | Fragment#onAttach()    | Fragment#onDestroy()    |
| ViewComponent             | View#super()           | 视图销毁时              |
| ViewWithFragmentComponent | View#super()           | 视图销毁时              |
| ServiceComponent          | Service#onCreate()     | Service#onDestroy()     |

### 3、组件的作用域

默认情况下，Hilt 中的所有绑定都未限制作用域。这就是说：每当应用请求绑定的时候，都会创建所需要的类型的一个实例。

Hilt 允许将绑定的作用域限定为特定组件。Hilt 为在同一个作用域限定范围内的组件创建一次实例，这样应用的绑定请求共享同一实例。

下面看看生成组件的作用域:

| Android 类                                 | 生成的组件                | 作用域                 |
| ------------------------------------------ | ------------------------- | ---------------------- |
| Application                                | ApplicationComponent      | @Singleton             |
| View Model                                 | ActivityRetainedComponent | @ActivityRetainedScope |
| Activity                                   | ActivityComponent         | @ActivityScoped        |
| Fragment                                   | FragmentComponent         | @FragmentScoped        |
| View                                       | ViewComponent             | @ViewScoped            |
| 带有 `@WithFragmentBindings` 注释的 `View` | ViewWithFragmentComponent | @ViewScoped            |
| Service                                    | ServiceComponent          | @ServiceScoped         |

作用域直接作用在对应的类上面就可以了。

### 4、使用步骤

#### 1）添加依赖

从下面的依赖可以看到使用了 annotationProcessor 注解处理器，说明 Hilt 中使用了 **APT 技术**

```
  //hilt 依赖
  implementation 'com.google.dagger:hilt-android:2.38.1'
  annotationProcessor  'com.google.dagger:hilt-compiler:2.38.1'
```

#### 2）添加插件

Hilt 插件是为了更好的使用 Hilt API , 提供的一个字节码转换器，可想而知这其中使用了**字节码插桩技术**。

```
//工程的build.gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.38.1'
    }
}

//模块module的build.gradle 添加应用插件
plugins {
    id 'dagger.hilt.android.plugin'
    id 'kotlin-kapt'
}
```

#### 3）Hilt 入口：自定义 application

每一个使用了 Hilt 的 app 都必须使用 @HiltAndroidApp 注解了的自定义 application , 这里是生成代码的入口位置，方便访问所有使用了 Dagger 的 module.

```
@HiltAndroidApp
class CustomApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        //这里可以处理一些全局的Component等
    }
}
```

#### 4）创建一个 Module

@InstallIn(ActivityComponent::class) 这个安装注解，我们需要特别注意一下：

1、他的参数必须是使用 @DefineComponent 注解的 Component ；

下面是笔者传递了 MainActivity 参数后编译出现的错误，MainActivity 里面是需要 Inject 该 Module 的

```
@InstallIn, can only be used with @DefineComponent-annotated classes,
but found: [com.leon.study_jetpack_hilt.MainActivity]
```

2、必须通过参数指明是在哪个 Component 上，不可不传参数；

下面是不传递参数，报错的异常错误：

```
Execution failed for task ':app:kaptDebugKotlin'.
> A failure occurred while executing org.jetbrains.kotlin.gradle.internal.KaptExecution
   > java.lang.reflect.InvocationTargetException (no error message)
```

```
@Module
@InstallIn(ActivityComponent::class)
class HttpResponseModule {

    @Provides
    fun getHttpModule(): HttpResponseModule {
        return HttpResponseModule()
    }
}
```

#### 5）添加注入点、Inject Modules

```
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var mHttpResponseModule:HttpResponseModule

    @Inject
    lateinit var mHttpResponseModule2:HttpResponseModule

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        println("HttpResponseModule object that via inject of hilt: hashcode = "+mHttpResponseModule.hashCode())

        println("HttpResponseModule object that via inject of hilt: hashcode = "+mHttpResponseModule2.hashCode())
    }
}

/*
* 输出的结果：
2021-08-06 14:33:39.641 27582-27582/com.leon.study_jetpack_hilt I/System.out: HttpResponseModule object that via inject of hilt: hashcode = 26182251
2021-08-06 14:33:39.641 27582-27582/com.leon.study_jetpack_hilt I/System.out: HttpResponseModule object that via inject of hilt: hashcode = 228151752
*/
```

### 5、接口实例注入

接口没有办法直接构造对象，那么就需要一个带有 @Binds 注解的抽象方法的 Module 来告知，该抽象方法的参数会告诉 Hilt 创建的实例的类型，而返回值的类型则告知是返回的哪个接口的实例。

下面的 java 代码是成功的：

```java
//定义接口
public interface IBook {
}

//接口的实现类
public class KotlinBook implements  IBook{

    @Inject
    KotlinBook(){

    }
}

//定义module
@Module
@InstallIn(ActivityComponent.class)
public abstract class BookModule {//这里注意是抽象类
    @Binds
    abstract IBook getBooks(KotlinBook book);//这里需要特别注意参数和返回值，方法是抽象方法
}

@AndroidEntryPoint
public class MainActivity extends AppCompatActivity {

    @Inject
    IBook book;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.e(TAG, "onCreate: "+book.hashCode());
    }
}
```

```java
kotlin代码：一直无法inject成功：
```

```
interface Book {
}

class JavaBook @Inject constructor() : Book {
}

/**
 * 模块必须是抽象类
 */
@Module
@InstallIn(ActivityComponent::class)
abstract class BookModule{
    //抽象函数
    @Binds
    abstract fun getBook(book: JavaBook): Book
}


@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var book: Book

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        println("HttpResponseModule object that via inject of hilt: listener----hashcode = " + book.hashCode())
    }
}
```

## 三、原理及生成源码分析

Hilt 的核心还是 Dagger 的原理来完成，他是在 Dagger 的基础上去掉了 Component 组件，而在编译的时候会根据 @InstallIn 注解来自动生成 Component，并把他们注入到相关的代码中，这样就可以全身心的只关注对象的创建和要把这些对象注入的位置。

Hilt 内置了针对 Android 类的一些 Component 组件，但是对于我们自己使用的是非 Android 内置类的时候，还是和 Dagger 的使用一样来做。

先来找找编译后生成的文件：

![img](http://upload-images.jianshu.io/upload_images/4753251-53cd890352e73393.png)

#### 1) Inject 流程

1. #### Hilt_MainActivity.java 源码：AndroidEntryPoint 该注解生成的 class 文件

   ```
   /**
    * A generated base class to be extended by the @dagger.hilt.android.AndroidEntryPoint annotated class. If using the Gradle plugin, this is swapped as the base class via bytecode transformation.
    */
   public abstract class Hilt_MainActivity extends AppCompatActivity implements GeneratedComponentManagerHolder {
     private volatile ActivityComponentManager componentManager;
   
      //这个方法调用了injectMainActivity() 
     protected void inject() {
       if (!injected) {
         injected = true;
         ((MainActivity_GeneratedInjector) this.generatedComponent()).injectMainActivity(UnsafeCasts.<MainActivity>unsafeCast(this));
       }
     }
   }
   ```

1. #### MainActivity_GeneratedInjector.java 源码：

   其实就是我们自己写的 Component，只不过不需要我们写，而是编译期帮我们生成了。

   ```
   @GeneratedEntryPoint
   @InstallIn(ActivityComponent.class)
   public interface MainActivity_GeneratedInjector {
     void injectMainActivity(MainActivity mainActivity);
   }
   ```

1. #### MainActivity_GeneratedInjector 这个接口的实现在 CustomApplication_HiltComponents.java 文件中的一个静态抽象类：

   ```
    @ActivityScoped
     public abstract static class ActivityC implements MainActivity_GeneratedInjector,
         ActivityComponent,
         DefaultViewModelFactories.ActivityEntryPoint,
         HiltWrapper_HiltViewModelFactory_ActivityCreatorEntryPoint,
         FragmentComponentManager.FragmentComponentBuilderEntryPoint,
         ViewComponentManager.ViewComponentBuilderEntryPoint,
         GeneratedComponent {
       @Subcomponent.Builder
       abstract interface Builder extends ActivityComponentBuilder {
       }
     }
   ```

1. #### 在 DaggerCustomApplication_HiltComponents_SingletonC.java 文件中看看 ActvityC 的实例类：activityCImpl

   ```
   private static final class ActivityCImpl extends CustomApplication_HiltComponents.ActivityC {
       private final HttpResponseModule httpResponseModule;
       //这个我们在注册流程里面来看
       private final DaggerCustomApplication_HiltComponents_SingletonC singletonC;
   
       private ActivityCImpl(DaggerCustomApplication_HiltComponents_SingletonC singletonC,
           ActivityRetainedCImpl activityRetainedCImpl, HttpResponseModule httpResponseModuleParam,
           Activity activityParam) {
         this.singletonC = singletonC;
         this.activityRetainedCImpl = activityRetainedCImpl;
         this.httpResponseModule = httpResponseModuleParam;
   
       }
       
       @Override
       public void injectMainActivity(MainActivity mainActivity) {
         injectMainActivity2(mainActivity);
       }
   
       private MainActivity injectMainActivity2(MainActivity instance) {
         MainActivity_MembersInjector.injectMHttpResponseModule(instance, HttpResponseModule_GetHttpModuleFactory.getHttpModule(httpResponseModule));
         MainActivity_MembersInjector.injectMHttpResponseModule2(instance, HttpResponseModule_GetHttpModuleFactory.getHttpModule(httpResponseModule));
         return instance;
       }
     }
   ```

1. #### 继续进入 MainActivity_MembersInjector.java 文件：

   看到这个文件就都明了了，这就是 Dagger 里面 Inject 生成的代码。

   

   ```
   public final class MainActivity_MembersInjector implements MembersInjector<MainActivity> {
     private final Provider<HttpResponseModule> mHttpResponseModuleProvider;
   
     private final Provider<HttpResponseModule> mHttpResponseModule2Provider;
   
     public MainActivity_MembersInjector(Provider<HttpResponseModule> mHttpResponseModuleProvider,
         Provider<HttpResponseModule> mHttpResponseModule2Provider) {
       this.mHttpResponseModuleProvider = mHttpResponseModuleProvider;
       this.mHttpResponseModule2Provider = mHttpResponseModule2Provider;
     }
   
     public static MembersInjector<MainActivity> create(
         Provider<HttpResponseModule> mHttpResponseModuleProvider,
         Provider<HttpResponseModule> mHttpResponseModule2Provider) {
       return new MainActivity_MembersInjector(mHttpResponseModuleProvider, mHttpResponseModule2Provider);
     }
   
     @Override
     public void injectMembers(MainActivity instance) {
       injectMHttpResponseModule(instance, mHttpResponseModuleProvider.get());
       injectMHttpResponseModule2(instance, mHttpResponseModule2Provider.get());
     }
   
     @InjectedFieldSignature("com.leon.study_jetpack_hilt.MainActivity.mHttpResponseModule")
     public static void injectMHttpResponseModule(MainActivity instance,
         HttpResponseModule mHttpResponseModule) {
       instance.mHttpResponseModule = mHttpResponseModule;
     }
   
     @InjectedFieldSignature("com.leon.study_jetpack_hilt.MainActivity.mHttpResponseModule2")
     public static void injectMHttpResponseModule2(MainActivity instance,
         HttpResponseModule mHttpResponseModule2) {
       instance.mHttpResponseModule2 = mHttpResponseModule2;
     }
   }
   ```

#### 2) 注册流程

1. #### Hilt_CustomApplication.java

   这里主要就是创建了一个 DaggerCustomApplication_HiltComponents_SingletonC 的实例，并与当前 app 的应用上下文绑定；

   

   ```
   public abstract class Hilt_CustomApplication extends Application implements GeneratedComponentManagerHolder {
     private final ApplicationComponentManager componentManager = new ApplicationComponentManager(new ComponentSupplier() {
       @Override
       public Object get() {
         return DaggerCustomApplication_HiltComponents_SingletonC.builder()
             .applicationContextModule(new ApplicationContextModule(Hilt_CustomApplication.this))
             .build();
       }
     });
   }
   
   
   //ApplicationContextModule  类的实现
   @Module
   @InstallIn(SingletonComponent.class)
   public final class ApplicationContextModule {
     private final Context applicationContext;
   
     public ApplicationContextModule(Context applicationContext) {
       this.applicationContext = applicationContext;
     }
   
     @Provides
     @ApplicationContext
     Context provideContext() {
       return applicationContext;
     }
   
     @Provides
     Application provideApplication() {
       return Contexts.getApplication(applicationContext);
     }
   }
   ```

1. #### DaggerCustomApplication_HiltComponents_SingletonC .java

先来看看该类的结构：大量的静态内部类

![img](http://upload-images.jianshu.io/upload_images/4753251-90d387a094d91f7a.png)

image-20210806170025050.png

其实就是将 app 的上下文保存下来，并创建了自己的实例：

```
public final class DaggerCustomApplication_HiltComponents_SingletonC extends CustomApplication_HiltComponents.SingletonC {
  private final ApplicationContextModule applicationContextModule;

  private final DaggerCustomApplication_HiltComponents_SingletonC singletonC = this;

  private DaggerCustomApplication_HiltComponents_SingletonC(
      ApplicationContextModule applicationContextModuleParam) {
    this.applicationContextModule = applicationContextModuleParam;

  }

  public static Builder builder() {
    return new Builder();
  }

  @Override
  public void injectCustomApplication(CustomApplication customApplication) {
  }

  @Override
  public ActivityRetainedComponentBuilder retainedComponentBuilder() {
    return new ActivityRetainedCBuilder(singletonC);
  }

  @Override
  public ServiceComponentBuilder serviceComponentBuilder() {
    return new ServiceCBuilder(singletonC);
  }
    
    //静态内部类，来创建DaggerCustomApplication_HiltComponents_SingletonC的实例
    public static final class Builder {
    private ApplicationContextModule applicationContextModule;

    private Builder() {
    }

    public Builder applicationContextModule(ApplicationContextModule applicationContextModule) {
      this.applicationContextModule = Preconditions.checkNotNull(applicationContextModule);
      return this;
    }

    public CustomApplication_HiltComponents.SingletonC build() {
      Preconditions.checkBuilderRequirement(applicationContextModule, ApplicationContextModule.class);
      return new DaggerCustomApplication_HiltComponents_SingletonC(applicationContextModule);
    }
  }
 }
```

1. #### ActivityRetainedBuilder 这个类实现，其实就是创了 ActivityRetainedCImpl 的实例对象