Android 模块开发之 SPI - 简书

Java 提供的 SPI 全名就是 Service Provider Interface, 下面是一段官方的解释,，其实就是为某个接口寻找服务的机制，有点类似 IOC 的思想，将装配的控制权移交给 Servic...

Java 提供的 SPI 全名就是 Service Provider Interface, 下面是一段官方的解释,，***其实就是为某个接口寻找服务的机制，有点类似 IOC 的思想，将装配的控制权移交给 ServiceLoader\***。SPI 在平时我们用到的会比较少，但是在 Android 模块开发中就会比较有用，不同的模块可以基于接口编程，每个模块有不同的实现 service provider, 然后通过 SPI 机制自动注册到一个配置文件中，就可以实现在程序运行时扫描加载同一接口的不同 service provider。这样模块之间不会基于实现类硬编码，可插拔。

```
* <p> A <i>service</i> is a well-known set of interfaces and (usually
* abstract) classes.  A <i>service provider</i> is a specific implementation
* of a service.  The classes in a provider typically implement the interfaces
* and subclass the classes defined in the service itself. 

* <p> For the purpose of loading, a service is represented by a single type,
* that is, a single interface or abstract class.  (A concrete class can be
* used, but this is not recommended.)  A provider of a given service contains
* one or more concrete classes that extend this <i>service type</i> with data
* and code specific to the provider. 
```

说的可能比较抽象，我们还是通过一个栗子来看看 SPI 在 Android 中的应用。先看张工程结构图，有四个模块，一个是 app 主程序模块，有一个接口 interface 模块，另外两个就是接口的不同实现模块 adisplay 和 bdisplay。



![img](http://upload-images.jianshu.io/upload_images/2608779-ff9c7b7b5741ec62.png)



工程结构. png

栗子很简单，就是点击按钮，加载不同模块实现的方法，如下图所示：



![img](http://upload-images.jianshu.io/upload_images/2608779-c7bb98cfd269c9f1.png)



栗子. png

### 1. 接口

首先就是模块之间通信接口的实现，我们这里单独抽出一个模块 interface，后面接口的不同实现模块可以都依赖同一个接口模块。接口里面就是一个简单的接口：

```java
package com.example;

public interface Display {
    String display();
}
```

### 2. 模块实现

接下来就是用不同的模块实现接口，首先需要在模块的 build.gradle 中加入接口的依赖：

```java
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile project(':interface')
}
```

然后一个简单的实现类实现接口 Display。adisplay 模块的实现类就是 ADisplay。

```
package com.example;

public class ADispaly implements Display{
    @Override
    public String display() {
        return "This is display in module adisplay";
    }
}
```

bdisplay 模块的实现类就是 BDisplay。

```
package com.example;

public class BDisplay implements Display{
    @Override
    public String display() {
        return "This is display in module bdisplay";
    }
}
```

接着就是要把这些接口实现类注册到配置文件中，spi 的机制就是注册到 META-INF/services 中。***首先在 java 的同级目录中 new 一个包目录 resources，然后在 resources 新建一个目录 META-INF/services，再新建一个 file，file 的名称就是接口的全限定名，在我们的栗子中就是：com.example.Display，文件中就是不同实现类的全限定名，不同实现类分别一行。\***
adisplay 模块最后的工程结构下图所示。文件`com.example.Display`中的内容就是 ADispaly 实现类的全限定名`com.example.ADispaly`



![img](http://upload-images.jianshu.io/upload_images/2608779-9096d6fcc8692872.png)



模块工程结构. png

bdisplay 模块最后的工程结构和上图类似，文件`com.example.Display`中的内容就是 BDisplay 实现类的全限定名`com.example.BDisplay`。
模块的实现就是上面这些，接下来看下主程序模块 app 中有哪些步骤。

### 3. 加载不同服务

当然在主程序模块中也可以有自己的接口实现：

```
package com.example.juexingzhe.spidemo;

import com.example.Display;

public class DisplayImpl implements Display {
    @Override
    public String display() {
        return "This is display in module app";
    }
}
```

在配置文件`com.example.Display`中的内容就是`com.example.juexingzhe.spidemo.DisplayImpl`
在界面上放置一个按钮，点击按钮会记载所有模块配置文件中的实现类：

```
mButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                loadModule();
            }
});
```

关键的代码其实就是下面三行，***通过 ServiceLoader 来加载接口的不同实现类，然后会得到迭代器，在迭代器中可以拿到不同实现类全限定名，然后通过反射动态加载实例就可以调用 display 方法了。\***

```
ServiceLoader<Display> loader = ServiceLoader.load(Display.class);
mIterator = loader.iterator();
while (mIterator.hasNext()){
      mIterator.next().display();
}
```

### 4. 源码分析

主要工作都是在 ServiceLoader 中，这个类在 java.util 包中。先看下几个重要的成员变量, PREFIX 就是配置文件所在的包目录路径；service 就是接口名称，在我们这个例子中就是 Display；loader 就是类加载器，其实最终都是通过反射加载实例；providers 就是不同实现类的缓存，key 就是实现类的全限定名，value 就是实现类的实例；lookupIterator 就是内部类 LazyIterator 的实例。

```
    private static final String PREFIX = "META-INF/services/";

    
    private Class<S> service;

    
    private ClassLoader loader;

    
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    
    private LazyIterator lookupIterator;
```

上面提到 SPI 的三个关键步骤：

```
ServiceLoader<Display> loader = ServiceLoader.load(Display.class); mIterator = loader.iterator(); while (mIterator.hasNext()){ mIterator.next().display(); }
```

先看下第一个步骤 load：ServiceLoader 提供了两个静态的 load 方法, 如果我们没有传入类加载器，ServiceLoader 会自动为我们获得一个当前线程的类加载器，最终都是调用构造函数。

```
public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl =Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service,ClassLoader loader)
{
        return new ServiceLoader<>(service, loader);
}
```

在构造函数中工作很简单就是清除实现类的缓存，实例化迭代器。

```
public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = svc;
    loader = cl;
    reload();
}

private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
}
```

***注意了，我们在外面通过 ServiceLoader.load(Display.class) 并不会去加载 service provider，也就是懒加载的设计模式，这也是 Java SPI 的设计亮点。\***

那么 service provider 在什么地方进行加载？我们接着看第二个步骤`loader.iterator()`, 其实就是返回一个迭代器。我们看下官方文档的解释, 这个就是懒加载实现的地方，首先会到 providers 中去查找有没有存在的实例，有就直接返回，没有再到 LazyIterator 中查找。

```
 * Lazily loads the available providers of this loader's service. 
 *
 * <p> The iterator returned by this method first yields all of the
 * elements of the provider cache, in instantiation order.  It then lazily
 * loads and instantiates any remaining providers, adding each one to the
 * cache in turn.
public Iterator<S> iterator() {
        return new Iterator<S>() {

            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();

            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                return lookupIterator.hasNext();
            }

            public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
}
```

我们一开始 providers 中肯定是没有缓存的实例的，接着会到 LazyIterator 中查找，去看看 LazyIterator, 先看下 hasNext() 方法。

首先拿到配置文件名 fullName, 我们这个例子中是`com.example.Display`
\2. 通过类加载器获得所有模块的配置文件`Enumeration<URL> configs configs`
\3. 依次扫描每个配置文件的内容，返回配置文件内容`Iterator<String> pending`，每个配置文件中可能有多个实现类的全限定名，所以 pending 也是个迭代器。

```
public boolean hasNext() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    nextName = pending.next();
    return true;
}
```

在上面 hasNext() 方法中拿到的 nextName 就是实现类的全限定名，接下来我们去看看具体实例化工作的地方 next():

\1. 首先根据 nextName，Class.forName 加载拿到具体实现类的 class 对象
2.Class.newInstance() 实例化拿到具体实现类的实例对象
\3. 将实例对象转换 service.cast 为接口
\4. 将实例对象放到缓存中，providers.put(cn, p)，key 就是实现类的全限定名，value 是实例对象。
\5. 返回实例对象

```
public S next() {
    if (!hasNext()) {
        throw new NoSuchElementException();
    }
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found", x);
    }
    if (!service.isAssignableFrom(c)) {
        ClassCastException cce = new ClassCastException(
                service.getCanonicalName() + " is not assignable from " + c.getCanonicalName());
        fail(service,
             "Provider " + cn  + " not a subtype", cce);
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated: " + x,
             x);
    }
    throw new Error();          
}
```

到这里我们的源码分析之旅就结束了，主要思想其实就是懒加载的思想。

### 5. 总结

通过 Java 的 SPI 机制也有一点缺点就是在运行时通过反射加载类实例，这个对性能会有点影响。但是瑕不掩瑜，SPI 机制可以实现不同模块之间方便的面向接口编程，拒绝了硬编码的方式，解耦效果很好。用起来也简单，只需要在目录 META-INF/services 中配置实现类就行。源码中也用来了懒加载的思想，开发中可以借鉴。