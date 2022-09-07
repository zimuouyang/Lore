
## 1. Data Binding 介绍
Data Binding不算特别新的东西，2015 年 Google 就推出了，但即便是现在，很多人都没有学习过它，我就是这些工程师中的一位，因为我觉得 MVP 已经足够帮我处理日常的业务，Android Jetpack的出现，是我研究Data Binding的一个契机。

在进行下文之前，我有必要声明一下，MVVM和Data Binding是两个不同的概念，MVVM 是一种架构模式，而 Data Binding 是一个实现数据和 UI 绑定的框架，是构建 MVVM 模式的一个工具。

### 2.1 学习姿势
我依然认为官方文档是最好的学习途径：
官方文档：Data Binding Library
谷歌实验室：官方教程
官方 Demo 地址：https://github.com/googlecodelabs/android-databinding

## 二、实战
在这里，我打算先在上一节即学即用 Android Jetpack - Navigation 的基础代码上进行拓展（如有涉及到Navigation的代码，我会注明），本文会在登录和注册模块的基础上进行讲解，后期如有需要，会拓展到其他模块。

效果图，和之前的有点不一样：


Data Binding
### 第一步 在 app 模块下的build.gradle文件添加内容
``` gradle
android {
...
    dataBinding {
       enabled true
    }
}
``` 
### 第二步 构建 LoginModel
创建登录的LoginModel，LoginModel主要负责登录逻辑的处理以及两个输入框内容改变的时候数据更新的处理：
``` kotlin
class LoginModel constructor(name: String, pwd: String, context: Context) {
    val n = ObservableField<String>(name)
    val p = ObservableField<String>(pwd)
    var context: Context = context

    /**
     * 用户名改变回调的函数
     */
    fun onNameChanged(s: CharSequence) {
        n.set(s.toString())
    }

    /**
     * 密码改变的回调函数
     */
    fun onPwdChanged(s: CharSequence, start: Int, before: Int, count: Int) {
        p.set(s.toString())
    }

    fun login() {
        if (n.get().equals(BaseConstant.USER_NAME)
            && p.get().equals(BaseConstant.USER_PWD)
        ) {
            Toast.makeText(context, "账号密码正确", Toast.LENGTH_SHORT).show()
            val intent = Intent(context, MainActivity::class.java)
            context.startActivity(intent)
        }
    }
}
``` 
我相信同学们可能会对ObservableField存在疑惑，那么ObservableField是什么呢？它其实是一个可观察的域，通过泛型来使用，可以使用的方法也就三个：

方法	作用
ObservableField(T value)	构造函数，设置可观察的域
T get()	获取可观察的域的内容，可以使用 UI 控件监测它的值
set(T value)	设置可观察的域，设置成功之后，会通知 UI 控件进行更新
不过，除了使用ObservableField之外，Data Binding为我们提供了基本类型的ObservableXXX(如ObservableInt) 和存放容器的ObservableXXX(如ObservableList<T>) 等，同样，如果你想让你自定义的类变成可观察状态，需要实现Observable接口。

我们再回头看看LoginModel这个类，它其实只有分别用来观察name和pwd的成员变量n和p，外加一个处理登录逻辑的方法，非常简单。

## 第三步 创建布局文件
引入Data Binding之后的布局文件的使用方式会和以前的布局使用方式有很大的不同，且听我一一解释：
***
| 标签名 | 作用 |
| :-----| ----: |
| layout | 用作布局的根节点，只能包裹一个 View 标签，且不能包裹 merge 标签。 |
| data | Data Binding 的数据，只能存在一个 data 标签。 |
| variable | data中使用，数据的变量标签，type属性指明变量的类，如com.joe.jetpackdemo.viewmodel.LoginModel。name属性指明变量的名字，方便布局中使用。 |
| import | data中使用，需要使用静态方法和静态常量，如需要使用 View.Visble 属性的时候，则需导入<import type="android.view.View"/>。type属性指明类的路径，如果两个import标签导入的类名相同，则可以使用alias属性声明别名，使用的时候直接使用别名即可。 |	
| include | View 标签中使用，作用同普通布局中的include一样，需要使用bind:<参数名>传递参数|


``` xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools">

    <data>
        <!--需要的viewModel,通过mBinding.vm=mViewMode注入-->
        <variable
            name="model"
            type="com.joe.jetpackdemo.viewmodel.LoginModel"/>

        <variable
            name="activity"
            type="androidx.fragment.app.FragmentActivity"/>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:id="@+id/txt_cancel"
            android:onClick="@{()-> activity.onBackPressed()}"
            />

        <TextView
            android:id="@+id/txt_title"
            app:layout_constraintTop_toTopOf="parent"
            .../>

        <EditText
            android:id="@+id/et_account"
            android:text="@{model.n.get()}"
            android:onTextChanged="@{(text, start, before, count)->model.onNameChanged(text)}"
            ...
            />

        <EditText
            android:id="@+id/et_pwd"
            android:text="@{model.p.get()}"
            android:onTextChanged="@{model::onPwdChanged}"
            ...
            />

        <Button
            android:id="@+id/btn_login"
            android:text="Sign in"
            android:onClick="@{() -> model.login()}"
            android:enabled="@{(model.p.get().isEmpty()||model.n.get().isEmpty()) ? false : true}"
            .../>


    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
``` 
variable有两个:

* model：类型为com.joe.jetpackdemo.viewmodel.LoginModel，
* 绑定用户名详见et_accountEditText 中的android:text="@{model.n.get()}"，当 EditText 输入框内容变化的时候有如下处理android:onTextChanged="@{(text, start, before, count)->model.onNameChanged(text)}"，
* 以及登录按钮处理android:onClick="@{() -> model.login()}"。
activity：类型为androidx.fragment.app.FragmentActivity，主要用来返回按钮的事件处理，详见txt_cancelTextView 的android:onClick="@{()-> activity.onBackPressed()}"。
对于以上的内容，我仍然有知识点需要讲解：

1. 属性的引用
如果想使用 ViewModel 中成员变量，如直接使用model.p。

2. 事件绑定
事件绑定包括方法引用和监听绑定：

方法引用：参数类型和返回类型要一致，参考et_pwdEditText 的android:onTextChanged引用。
监听绑定：相比较于方法引用，监听绑定的要求就没那么高了，我们可以使用自行定义的函数，参考et_accountEditText 的android:onTextChanged引用。

3. 表达式
如果你注意到了btn_loginButton 在密码没有内容的时候是灰色的：

LoginFragment
是因为它在android:enabled使用了表达式：@{(model.p.get().isEmpty()||model.n.get().isEmpty()) ? false : true}，它的意思是用户名和密码为空的时候登录的enable属性为 false，这是普通的三元表达式，除了上述的||和三元表达式之外，Data Binding还支持：
```
运算符 + - / * %
字符串连接 +
逻辑与或 && ||
二进制 & | ^
一元 + - ! ~
移位 >> >>> <<
比较 == > <>= <= (Note that < needs to be escaped as <)
instanceof
Grouping ()
Literals - character, String, numeric, null
Cast
方法调用
域访问
数组访问
三元操作符
除了上述之外，Data Binding新增了空合并操作符??，例如android:text="@{user.displayName ?? user.lastName}"，它等价于android:text="@{user.displayName != null ? user.displayName : user.lastName}"。
```
## 第四步 生成绑定类
我们的布局文件创建完毕之后，点击Build下面的Make Project，让系统帮我生成绑定类，生成绑定的类如下：

生成的绑定类
下面我们只需在

LoginFragment
完成绑定即可，绑定操作既可以使用上述生成的

FragmentLoginBinding
也可以使用自带的

DataBindingUtil
完成：

### 1. 使用 DataBindingUtil
我们可以看一下DataBindingUtil的一些常用 Api：

函数名	作用
setContentView	用来进行 Activity 下面的绑定
inflate	用来进行 Fragment 下面的绑定
bind	用来进行 View 的绑定
LoginFragment绑定代码如下：
``` kotlin
override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val binding: FragmentLoginBinding = DataBindingUtil.inflate(
            inflater
            , R.layout.fragment_login
            , container
            , false
        )
        loginModel = LoginModel("","",context!!)
        binding.model = loginModel
        binding.activity = activity
        return binding.root
    }
``` 
### 2. 使用生成的FragmentLoginBinding
使用方法与第一种类似，仅需将生成方式改成val binding = FragmentLoginBinding.inflate( inflater , container , false )即可

运行一下代码，开始图的效果就出现了。

## 三、更多
Data Binding还有一些有趣的功能，为了让同学们了解到更多的知识，我们在这里有必要探讨一下：

### 1. 布局中属性的设置
#### 1.1 有属性有 setter 的情况
如果 XXXView 类有成员变量borderColor，并且 XXXView 类有setBoderColor(int color)方法，那么在布局中我们就可以借助Data Binding直接使用app:borderColor这个属性，不太明白？没关系，以DrawerLayout为例，DrawerLayout没有声明app:scrimColor、app:drawerListener，但是DrawerLayout有mScrimColor:int、mListener:DrawerListener这两个成员变量并且具有这两个属性的setter的方法，他就可以直接使用app:scrimColor、app:drawerListener这两个属性，代码如下：
``` xml
<android.support.v4.widget.DrawerLayout
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:scrimColor="@{@color/scrim}"
    app:drawerListener="@{fragment.drawerListener}">
 ```      
#### 1.2 没有 setter 但是有相关方法
还用 XXXView 为例，它有成员变量borderColor，这次设置borderColor的方法是setBColor(总有程序员乱写方法名~)，强行用app:borderColor显然是行不通的，可以这样用的前提是必须有setBoderColor(int color)方法，显然setBColor不匹配，但我们可以通过BindingMethods注解实现app:borderColor的使用，代码如下：
``` kotlin
@BindingMethods(value = [
    BindingMethod(
        type = 包名.XXXView::class,
        attribute = "app:borderColor",
        method = "setBColor")])
``` 
#### 1.3 自定义属性
这次不仅没setter方法，甚至连成员变量都需要自带（条件越来越刻苦~），这次我们的目标就是给 EditText 添加文本监听器，先在LoginModel中自定义一个监听器并使用@BindingAdapter注解：
``` kotlin
// SimpleWatcher 是简化了的TextWatcher
    val nameWatcher = object : SimpleWatcher() {
        override fun afterTextChanged(s: Editable) {
            super.afterTextChanged(s)

            n.set(s.toString())
        }
    }

    @BindingAdapter("addTextChangedListener")
    fun addTextChangedListener(editText: EditText, simpleWatcher: SimpleWatcher) {
        editText.addTextChangedListener(simpleWatcher)
    }
``` 
这样我们就可以在布局文件中对 EditText 愉快的使用app:addTextChangedListener属性了：
``` xml
<EditText
            android:id="@+id/et_account"
            android:text="@{model.n.get()}"
            app:addTextChangedListener="@{model.nameWatcher}"
            ...
            />
``` 
效果与我们之前使用的时候一样

### 2. 双向绑定
使用双向绑定可以简化我们的代码，比如我们上面的 EditText 在实现双向绑定之后既不需要添加SimpleWatcher也不需要用方法调用，怎么实现呢？代码如下：
``` xml
<EditText
            android:id="@+id/et_account"
            android:text="@={model.n.get()}"
            ...
            />
```
仅仅在将@{model.n.get()}替换为@={model.n.get()}，多了一个=号而已，需要注意的是，属性必须是可观察的，可以使用上面提到的ObservableField，也可以自定义实现BaseObservable接口，双向绑定的时候需要注意无限循环，更多关于双向绑定还请查看官方文档。

## 四、总结

Data Binding 总结
Data Binding
的介绍可能没有那么全面，基本使用没什么问题了，想要了解更多可以查看官方文档呦~，本人水平有限，难免理解有误差，欢迎指正。

