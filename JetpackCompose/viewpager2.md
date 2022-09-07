## ViewPager2 使用详解
ViewPager2 是 ViewPager 的升级版。ViewPager2 是基于 RecyclerView 实现的，在解决了很多使用 ViewPager 时遇到的问题的同时，还加入自己的一些新特性。下面我们来介绍他的使用。

### 一：ViewPager2 使用
1.1 添加依赖
在 app 的 build.gradle 里添加如下依赖

dependencies {
  implementation "androidx.viewpager2:viewpager2:1.0.0"
}
复制代码
1.2 ViewPager2 布局文件
``` xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/view_pager"
    android:layout_width="match_parent"
    android:layout_height="300dp"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintLeft_toLeftOf="parent"
    app:layout_constraintRight_toRightOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
```

</androidx.constraintlayout.widget.ConstraintLayout>
复制代码
### 1.3 ViewPager2 的 Adapter
因为 ViewPager2 是基于 RecyclerView 的，所以它使用的 Adapter 也是 RecyclerView 的 Adapter。所以自定义一个叫 MyAdapter 的来作为 ViewPager2 的 Adapter
``` kotlin
class MyAdapter :RecyclerView.Adapter<MyAdapter.MHolder>(){


private var mColorsList = ArrayList<Int>()

override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MHolder {
    val itemView = LayoutInflater.from(parent.context).inflate(R.layout.item_page,parent,false)
    return MHolder(itemView)
}

override fun onBindViewHolder(holder: MHolder, position: Int) {
    holder.view.setBackgroundColor(mColorsList[position])
}

override fun getItemCount(): Int {
    return mColorsList.size
}

fun setData(colors:List<Int>){
    mColorsList.clear()
    if(colors?.isNotEmpty()){
        mColorsList.addAll(colors)
    }
    notifyDataSetChanged()
}

class MHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    val view = itemView.findViewById<View>(R.id.view_item)
}
}
```

item_page.xml 的代码如下：
``` xml
<?xml version="1.0" encoding="utf-8"?>

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:orientation="vertical"
    android:layout_height="match_parent">
<View
    android:id="@+id/view_item"
    android:background="@color/colorAccent"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
    </LinearLayout>
```



根据传入的不同的颜色值去设置 View 的背景。

### 1.4 ViewPager2 的绑定 Adapter
``` kotlin
class ViewPager2TestActivity :AppCompatActivity(){

private val list = arrayListOf<Int>(Color.RED,Color.YELLOW,Color.BLUE,Color.GREEN)

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_viewpager2test)

    val adapter = MyAdapter()
    adapter.setData(list)
    view_pager.adapter = adapter
}
}
```



这样简单的 ViewPager2 的使用就完成了。

### 1.5 ViewPager2 支持竖直方向
除了传统的水平分页之外，ViewPager2 还支持垂直分页。您可以通过设置 ViewPager2 元素的 android:orientation 属性为其启用垂直分页：

``` xml
<androidx.viewpager2.widget.ViewPager2
   xmlns:android="http://schemas.android.com/apk/res/android"
   android:id="@+id/pager"
   android:orientation="vertical" />
```
您还可以使用 setOrientation() 方法，以编程方式设置此属性。
```
view_pager.orientation = ViewPager2.ORIENTATION_VERTICAL
```
### 1.6 ViewPager2 支持从右到左
ViewPager2 支持从右到左 (RTL) 分页。系统会根据语言区域在适当的情况下自动启用 RTL 分页，不过您也可以通过设置 ViewPager2 元素的 android:layoutDirection 属性为其手动启用 RTL 分页：
``` xml
<androidx.viewpager2.widget.ViewPager2
   xmlns:android="http://schemas.android.com/apk/res/android"
   android:id="@+id/pager"
   android:layoutDirection="rtl" />
```
您还可以使用 setLayoutDirection() 方法，以编程方式设置此属性。
``` kotlin
view_pager.layoutDirection = LayoutDirection.RTL
```
### 1.7 ViewPager2 滑动监听
``` kotlin
view_pager.registerOnPageChangeCallback(object :ViewPager2.OnPageChangeCallback(){
    override fun onPageSelected(position: Int) {
        super.onPageSelected(position)
        Toast.makeText(this@ViewPager2TestActivity, "ccm== $position", Toast.LENGTH_SHORT).show()
    }
})
```

### 1.8 setUserInputEnabled 禁止滑动
在使用 ViewPager 的时候想要禁止用户滑动需要重写 ViewPager 的 onInterceptTouchEvent。而 ViewPager2，我们可以直接通过 setUserInputEnabled 为 false 禁止滑动
``` kotlin
view_pager.isUserInputEnabled = false
```
### 1.9 setCurrentItem
切换到某个位置下，跟 ViewPager 一样，ViewPager2 也是通过 setCurrentItem 设置。
``` kotlin
view_pager.currentItem = 1
```

### 1.10 fakeDragBy
ViewPager2 新增了一个 fakeDragBy 的方法，我们可以通过 fakeDragBy 来移动，fakeDragBy(-10), 负数是说明向下一个页面，正数表示向前一个页面滑动，但使用 fakeDragBy 前需要先调用 beginFakeDrag 方法才能生效。可以使用 endFakeDrag 停止
``` kotlin
btn_fake.setOnClickListener {
   view_pager.beginFakeDrag()
   view_pager.fakeDragBy(-500f)
}
btn_fake.setOnClickListener {
   view_pager.beginFakeDrag()
   if(view_pager.fakeDragBy(-310f)) view_pager.endFakeDrag()
}
```

### 1.11 ViewPager2 的 offScreenPageLimit
offScreenPageLimit 在 ViewPager 中就已经存在，这个参数用来控制 ViewPager 左右两端预加载页面的个数。为了保证 ViewPager 的流畅性，offScreenPageLimit 被强制规定为大于 0 的数，即使我们将其设置为 0，ViewPager 内部也会将其改为 1。因此 ViewPager 就被强制左右两边至少加载一个页面。而在 ViewPager2 中针对这一问题做了优化。我们点开 ViewPager2 的源码来看下:

``` kotlin
public static final int OFFSCREEN_PAGE_LIMIT_DEFAULT = -1;
private @OffscreenPageLimit int mOffscreenPageLimit = OFFSCREEN_PAGE_LIMIT_DEFAULT;

/** @hide */
@SuppressWarnings("WeakerAccess")
@RestrictTo(LIBRARY_GROUP_PREFIX)
@Retention(SOURCE)
@IntDef({OFFSCREEN_PAGE_LIMIT_DEFAULT})
@IntRange(from = 1)
public @interface OffscreenPageLimit {}

public void setOffscreenPageLimit(@OffscreenPageLimit int limit) {
   if (limit < 1 && limit != OFFSCREEN_PAGE_LIMIT_DEFAULT) {
       throw new IllegalArgumentException(
                    "Offscreen page limit must be OFFSCREEN_PAGE_LIMIT_DEFAULT or a number > 0");
   }
   mOffscreenPageLimit = limit;
   // Trigger layout so prefetch happens through getExtraLayoutSize()
   mRecyclerView.requestLayout();
}
```
我们发现 ViewPager2 中 offScreenPageLimit 的默认值被设置为了 - 1，而当 offScreenPageLimit 为 - 1 的时候，使用的是 RecyclerView 的缓存机制。而当 offScreenPageLimit 大于 1 时，才会去实现预加载。 以前我们想要去实现 ViewPager 只初始化一个 Fragment 的时候，根据 Fragment 的生命周期去写了很多的代码去实现懒加载。现在使用 ViewPager2 不需要了，只要把 offScreenPageLimit 设置成 - 1 就可以了。ViewPager2 就会去使用 RecyclerView 的缓存机制了。

## 二：ViewPager2 的 PageTransformer
ViewPager2 的 Transformer 功能有了很大的扩展。ViewPager2 不仅可以通过 PageTransformer 用来设置页面动画，还可以用 PageTransformer 设置页面间距以及同时添加多个 PageTransformer。

### 2.1：ViewPager2 的给页面设置间距
ViewPager2 中可以通过 setPageTransformer 给页面之间设置间距
```
view_pager.setPageTransformer(MarginPageTransformer(30))
```
### 2.2：ViewPager2 的给页面之间设置跳转动画
先自定义一个 ScaleInTransformer 去实现 ViewPager2.PageTransformer
``` kotlin
class ScaleInTransformer :ViewPager2.PageTransformer{

private val minScale = 0.85f
private val centerF = 0.5f

override fun transformPage(page: View, position: Float) {
    page.elevation = -abs(position)
    val pageW = page.width
    val pageH = page.height
    page.pivotY = pageH/2f
    page.pivotX = pageW/2f

    if(position<-1){
        page.scaleX = minScale
        page.scaleY = minScale
        page.pivotX = pageW.toFloat()
    }else if(position<=1){
        if (position < 0) {
            val scaleFactor = (1 + position) * (1 - minScale) + minScale
            page.scaleX = scaleFactor
            page.scaleY = scaleFactor
            page.pivotX = pageW * (centerF + centerF * -position)
        } else {
            val scaleFactor = (1 - position) * (1 - minScale) + minScale
            page.scaleX = scaleFactor
            page.scaleY = scaleFactor
            page.pivotX = pageW * ((1 - position) * centerF)
        }
    }else{
        page.scaleX = minScale
        page.scaleY = minScale
        page.pivotX = 0f
    }
}
}

```

上面定义了一个缩放的动画，把他作为页面切换的时候的动画。再通过 CompositePageTransformer 的 addTransformer 方法添加页面切换动画，代码如下：
``` kotlin
val compositePageTransformer = CompositePageTransformer()
compositePageTransformer.addTransformer(ScaleInTransformer())
compositePageTransformer.addTransformer(MarginPageTransformer(40))
view_pager.setPageTransformer(compositePageTransformer)
```

### 2.3：ViewPager2 实现跨屏的效果
为了实现扩屏的效果，我们可以通过 padding 来实现。给 ViewPager2 里的 RecyclerView 设置 Padding
``` kotlin
view_pager.apply {
            offscreenPageLimit = 1
            val recyclerView = getChildAt(0) as RecyclerView
            recyclerView.apply {
                val padding = resources.getDimensionPixelOffset(R.dimen.dp_20)
                setPadding(padding,0,padding,0)
                clipToPadding = false
            }
        }

val compositePageTransformer = CompositePageTransformer()
compositePageTransformer.addTransformer(ScaleInTransformer())
compositePageTransformer.addTransformer(MarginPageTransformer(40))
view_pager.setPageTransformer(compositePageTransformer)
```

## 三：ViewPager2 跟 Fragment 的配合使用
ViewPager2 中新增的 FragmentStateAdapter 替代了 ViewPager 的 FragmentStatePagerAdapter 跟 FragmentPagerAdapter

### 3.1：实现 FragmentStateAdapter
``` kotlin
class HomeAdapterTest(activity: FragmentActivity) : FragmentStateAdapter(activity) {

companion object {
    const val HOME_COUNT = 3
    const val PAGE_ONE = 0
    const val PAGE_TWO = 1
    const val PAGE_THREE = 2
}

override fun createFragment(position: Int): Fragment {
    return when (position) {
        PAGE_ONE -> {
            FragmentTest1()
        }
        PAGE_TWO -> {
            FragmentTest2()
        }
        PAGE_THREE -> {
            FragmentTest3()
        }
        else -> FragmentTest1()
    }
}

override fun getItemCount(): Int {
    return HOME_COUNT
}
}
```


### 3.2：Activity 中的 ViewPager2
布局文件的代码如下：
``` xml
<?xml version="1.0" encoding="utf-8"?>

<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/view_pager"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    app:layout_constrainedHeight="true"
    app:layout_constraintBottom_toTopOf="@+id/btn_one"
    app:layout_constraintTop_toTopOf="parent" />

<Button
    android:id="@+id/btn_one"
    android:layout_width="0dp"
    android:layout_height="50dp"
    app:layout_constrainedWidth="true"
    android:text="切换到一"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintEnd_toStartOf="@+id/btn_two"
    app:layout_constraintStart_toStartOf="parent" />

<Button
    android:id="@+id/btn_two"
    android:layout_width="0dp"
    android:layout_height="50dp"
    android:text="切换到二"
    app:layout_constrainedWidth="true"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintEnd_toStartOf="@+id/btn_three"
    app:layout_constraintStart_toEndOf="@+id/btn_one" />

<Button
    android:id="@+id/btn_three"
    android:layout_width="0dp"
    android:layout_height="50dp"
    android:text="切换到三"
    app:layout_constrainedWidth="true"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintStart_toEndOf="@+id/btn_two" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

布局文件中存在一个 ViewPager2 和三个按钮，三个按钮点击分别可以切换到三个 Fragment，Activity 里的代码如下。
``` kotlin
class ViewPagerFragmentActivity :AppCompatActivity(),View.OnClickListener{
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_viewpager2fragment)
    btn_one.setOnClickListener(this)
    btn_two.setOnClickListener(this)
    btn_three.setOnClickListener(this)
    view_pager.adapter = HomeAdapterTest(this)
    view_pager.isUserInputEnabled = false
}

override fun onClick(v: View?) {
    when(v){
        btn_one->view_pager.setCurrentItem(0,false)
        btn_two->view_pager.setCurrentItem(1,false)
        btn_three->view_pager.setCurrentItem(2,false)
    }
}

}
```

view_pager.isUserInputEnabled = false 设置 ViewPager2 为不可滑动
view_pager.adapter = HomeAdapterTest(this) 设置 ViewPager2 的 Adapter 为 HomeAdapterTest
view_pager.setCurrentItem(0,false) 按钮监听事件里切换 viewPager2 的内容
## 四：ViewPager2 跟 TabLayout 的配合使用
由于需要使用到类 TabLayoutMediator，所以需要在 app 的 build.gradle 文件中添加如下的依赖：

implementation "com.google.android.material:material:1.1.0-beta01"

布局文件中的代码如下：
``` xml
<?xml version="1.0" encoding="utf-8"?>

<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


<com.google.android.material.tabs.TabLayout
    android:id="@+id/tab_layout"
    app:layout_constraintTop_toTopOf="parent"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>

<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/view_pager"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    app:layout_constrainedHeight="true"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintTop_toBottomOf="@+id/tab_layout" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

对应的 Activity 的代码
``` kotlin
view_pager.adapter = HomeAdapterTest(this)
view_pager.isUserInputEnabled = false
TabLayoutMediator(tab_layout,view_pager){
            tab, position ->
            tab.text = "Tab ${(position+1)}"
        }.attach()
```