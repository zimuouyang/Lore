在 jdk 1.2 以前，创建的对象只有处在可触及（reachaable）状态下，才能被程序所以使用，[垃圾回收](https://so.csdn.net/so/search?q=垃圾回收&spm=1001.2101.3001.7020)器一旦发现无用对象，便会对其进行回收。但是，在某些情况下，我们希望有些对象不需要立刻回收或者说从全局的角度来说并没有立刻回收的必要性。比如缓存系统的设计，在内存不吃紧或者说为了提高运行效率的情况下，一些暂时不用的对象仍然可放置在内存中，而不是立刻进行回收。因此，从 jdk 1.2 版本开始，java 设计人员把对象的引用细分为强引用（Strong Reference）、软引用 (Soft Reference)、弱引用(Weak Reference) 和虚引用 (Phantom Reference) 四种级别, 主要区别体现在在被 GC 回收的优先级上: 强引用 ->软引用 ->弱引用 ->虚引用。也就是说从 jdk 1.2 开始，垃圾回收器回收对象时，对象的可达性分析需要考虑考虑对象的引用强度，也就是说现在**对象的有效性 = 可达性 + 引用类型**。先来看下类层次结构如下：

![img](https://img-blog.csdn.net/20150620182522495?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGQ4NjQxNDAxMzA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

# 1. 引用的四种类型

## 1. 强引用（Strong Reference）

在代码中普遍使用的，类似`Person person=new Person();`如果一个对象具有强引用，则无论在什么情况下，GC 都不会回收被引用的对象。当**内存空间**不足时，JAVA 虚拟机宁可抛出 OutOfMemoryError 终止应用程序也不会回收具有强引用的对象。

## 2. 软引用（Soft Reference）

表示一个对象处在有用但非必须的状态。如果一个对象具有软引用，在内存空间充足时，GC 就不会回收该对象；当内存空间不足时，GC 会回收该对象的内存（回收发生在 OutOfMemoryError 之前）。

```
Person person=new Person();
SoftReference sr=new SoftReference(person);
```

软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被 GC 回收，Java 虚拟机就会把这个软引用加入到与之关联的引用队列中，以便在恰当的时候将该软引用回收。但是由于 GC 线程的优先级较低，通常手动调用 System.gc() 并不能立即执行 GC，因此弱引用所引用的对象并不一定会被马上回收。

## 3. 弱引用（Weak Reference）

用来描述非必须的对象。它类似软引用，但是强度比软引用更弱一些：弱引用具有更短的生命. GC 在扫描的过程中，一旦发现只具有被弱引用关联的对象，都会回收掉被弱引用关联的对象。换言之，无论当前内存是否紧缺，GC 都将回收被弱引用关联的对象。

```
Person person=new Person();
WeakReference wr=new WeakReference(person);
```

同样弱引用也可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被 GC 回收了，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中，以便在恰当的时候将该弱引用回收。

## 4. 虚引用（Phantom Reference）

虚引等同于没有引用，这意味着在任何时候都可能被 GC 回收，设置虚引用的目的是为了被虚引用关联的对象在被垃圾回收器回收时，能够收到一个系统通知。（被用来跟踪对象被 GC 回收的活动）虚引用和弱引用的区别在于：虚引用在使用时**必须**和引用队列（ReferenceQueue）联合使用，其在 GC 回收期间的活动如下：

```
ReferenceQueue queue=new ReferenceQueue();
PhantomReference pr=new PhantomReference(object.queue);
```

也即是 GC 在回收一个对象时，如果发现该对象具有虚引用，那么在回收之前会首先该对象的虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入虚引用来了解被引用的对象是否被 GC 回收。

# ReferenceQueue 和 Reference

## 1. ReferenceQueue 含义及作用

通常我们将其 ReferenceQueue 翻译为引用队列，换言之就是存放引用的队列，保存的是 Reference 对象。其作用在于 **Reference 对象所引用的对象**被 GC 回收时，该 Reference 对象将会被加入引用队列中（ReferenceQueue）的队列末尾, 这相当于是一种通知机制. 当关联的引用队列中有数据的时候，意味着引用指向的堆内存中的对象被回收。通过这种方式，[JVM](https://so.csdn.net/so/search?q=JVM&spm=1001.2101.3001.7020) 允许我们在对象被销毁后，做一些我们自己想做的事情。JVM 提供了一个 ReferenceHandler 线程，将引用加入到注册的引用队列中

ReferenceQueue 常用的方法：
`public Reference<? extends T> poll()`：从队列中取出一个元素，队列为空则返回 null；
`public Reference<? extends T> remove()`：从队列中出对一个元素，若没有则阻塞至有可出队元素；
`public Reference<? extends T> remove(long timeout)`：从队列中出对一个元素，若没有则阻塞至有可出对元素或阻塞至超过 timeout 毫秒；

见如下代码：

```
ReferenceQueue< Person> rq=new ReferenceQueue<Person>();
Person person=new Person();
SoftReference sr=new SoftReference(person,rq);
```

这段代码中，对于 Person 对象有两种引用类型，一是 person 的强引用，而是 sr 的软引用。sr 强引用了 SoftReference 对象，该对象软引用了 Person 对象。当 person 被回收时，sr 所强引用的对象将会被放到 rq 的队列末尾。利用 ReferenceQueue 可以清除失去了软引用对象的 SoftReference, 如下操作：

```
SoftReference ref=null;
while((ref=(Person)rq.poll())!=null){
    //清除
}
```

## 2. Reference 类

Reference 是 SoftReference，WeakReference,PhantomReference 类的父类，其内部通过一个 next 字段来构建了一个 Reference 类型的单向列表，而 queue 字段存放了引用对象对应的引用队列，若在 Reference 的子类构造函数中没有指定，则默认关联一个 ReferenceQueue.NULL 队列。

# 3. 四种引用类型使用场景

强引用类型是在代码中普遍存在，无须解释太多了
软引用和弱引用: 两者都可以实现缓存功能，但软引用实现的缓存通常用在服务端，而在移动设备中的内存更为紧缺，对垃圾回收更为敏感，因此 android 中的缓存通常是用弱引用来实现（比如 LruCache）
在开发中有这么一个场景：用户信息查询。在不考虑对用户信息更改的情况下，通常有以下两种方案来实现：
\1. 每次查询时，连接数据库获取信息，缺点是 IO 读频繁，平均响应时间较长。优点是内存占用少。
\2. 第一次查询时，读取数据库后将用户信息存放在内存中，以后每次查询从内存中读取，优点是读取速度快，在请求次数较少的情况下，内存占用较多。
现在我们用采用第二种方案，基于缓存来设计，代码如下：

User.java

```java
public class User {

    private String id;
    private String name;
    private int age;
    public User(String id) {
        super();
        this.id = id;
    }
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
    @Override
    public String toString() {
        return "User [id=" + id + ", name=" + name + ", age=" + age + "]";
    }
```

UserCache.java

```java
public class UserCache {
    private static UserCache cache;
    private Hashtable<String, UserRef> userRefs;
    private ReferenceQueue<User> q;

    private class UserRef extends SoftReference<User>{
        private String key="";
        public UserRef(User user,ReferenceQueue<User> q){
            super(user,q);
            key=user.getId();
        }
    }


    private UserCache(){
        userRefs=new Hashtable<>();
        q=new ReferenceQueue<>();
    }

    public static UserCache getInstance(){
        synchronized (UserCache.class) {
            if(cache==null){
                synchronized (UserCache.class) {
                    cache=new UserCache();
                }
            }

        }
        return cache;
    }

    /**
     * 缓存用户数据
     * @param user
     */
    private void cacheObject(User user){
        cleanCache();
        UserRef ref = new UserRef(user, q);
        userRefs.put(user.getId(), ref);
    }

    /**
     * 获取用户数据
     * @param id
     * @return
     */
    public User getObject(String id){
        User user=null;
        if(userRefs.containsKey(id)){
            System.err.println("get data from cache");
            UserRef ref = userRefs.get(id);
            user=ref.get();
        }
        if(user==null){
            System.err.println("get data from db");
            user=new User(id);
            user.setName("dong"+new Random().nextInt(10));
            cacheObject(user);
        }
        return user;
    }

    private void cleanCache() {
        UserRef ref=null;
        while((ref=(UserRef) q.poll())!=null){
            userRefs.remove(ref.key);
        }

    }

    public void clearAll(){
        cleanCache();
        userRefs.clear();
        System.gc();
        System.runFinalization();
    }

}
```

UserCacheTest.java

```java
public class UserCacheTest {

    public static void main(String[] args) {
        UserCache cache = UserCache.getInstance();

        for(int i=0;i<2;i++){
            System.out.println(cache.getObject("123").toString());
        }
        cache.clearAll();
        System.out.println(cache.getObject("123").toString());
    }
}
```

这样简单的缓存就完成了，输出结果如下：

get data from db
get data from cache
User [id=123, name=dong3, age=0]
User [id=123, name=dong3, age=0]
get data from db
User [id=123, name=dong2, age=0]

# 4. 到底是什么引用类型？



如果一个对象有多个引用类型，那在进行垃圾回收时如何判断对象的可达性呢？其原则如下：

单挑引用链的可达性以最弱的一个引用类型来决定；
多条引用链的可达性以最强的一个引用类型来决定；

举例说明，如下图所示：

![img](https://img-blog.csdn.net/20151117143806818)


对 Object 2 进行分析，路径 1-1——>2-1 中取最弱的引用，软引用；路径 1-2——>2-2 中取最弱引用，虚引用；在这两条路径中取最强引用，软引用。因此 Object 2 是最终的引用类型是软引用。

# 示例代码

我们通过一段代码帮助理解这四种：

```java
    private void test_gc1(){
        //在heap中创建内容为"wohenhao"的对象，并建立a到该对象的强引用，此时该对象时强可及
        String a=new String("wohenhao");
        //对heap中的对象建立软引用，此时heap中的对象仍然是强可及
        SoftReference< ?> softReference=new SoftReference<String>(a);
        //对heap中的对象建立弱引用，此时heap中的对象仍然是强可及
        WeakReference< ?> weakReference=new WeakReference<String>(a);
        System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
        //heap中的对象从强可及到软可及
        a=null;
        System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
        softReference.clear();//heap中对象从软可及变成弱可及,此时调用System.gc()，
        System.gc();
        System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
    }

    private void test_gc2(){
    //在heap中创建内容为"wohenhao"的对象，并建立a到该对象的强引用，此时该对象时强可及
        String a=new String("wohenhao");
        //对heap中的对象建立软引用，此时heap中的对象仍然是强可及
        SoftReference< ?> softReference=new SoftReference<String>(a);
        //对heap中的对象建立弱引用，此时heap中的对象仍然是强可及
        WeakReference< ?> weakReference=new WeakReference<String>(a);
        System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
        a=null;//heap中的对象从强可及到软可及
        System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
        System.gc();
        System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
    }
```