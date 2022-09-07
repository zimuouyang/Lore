
# Android 无所不能的 hook，让应用不再崩溃
之前推送了很多大厂分享，很多同学看完就觉得，大厂输出的理论知识居多，缺乏实践。

那这篇文章，我们将介绍一个大厂的库，这个库能够实打实的帮助大家解决一些问题。

今天的主角：初学者小张，资深研发老羊。

三方库中的 bug
这天 QA 上线前给小张反馈了一个 bug，应用启动就崩溃，小张一点不慌，插入 USB，触发，一看日志，原来是个空指针。

想了想，空指针比较好修复，大不了判空防御一下，于是回答：这个问题交给我，马上修复。

根据堆栈，找到了空指针的元凶。

忽然间，小张愣住了，这个空指针是个三方库在初始化的时候获取用户剪切板出错了。

这可怎么解决呢？

本来以为判个空防御一下完事，这会遇到硬茬了。

毕竟是自己装的逼，含着泪也要修复了，我们模拟下现场。
``` java
/**
 * 这是三方库中的调用
 */
public class Tools {
    
    public static String getClipBoardStr(Context context) {
        ClipboardManager clipboardManager = (ClipboardManager) context.getSystemService(Context.CLIPBOARD_SERVICE);
        ClipData primaryClip = clipboardManager.getPrimaryClip();
        // NPE
        ClipData.Item itemAt = primaryClip.getItemAt(0);
        if (itemAt == null) {
            return "";
        }
        CharSequence text = itemAt.getText();
        if (text == null) {
            return "";
        }
        return text.toString();
    }
}
``` 
我们写个按钮来触发一下：



果然发生了崩溃，空指针发生在clipboardManager.getPrimaryClip()，当手机上没有过复制内容时，getPrimaryClip返回的就是 null。

马上就要上线了，但是这个问题，也不是修复不了，根据自己的经验，大多数系统服务都可以被 hook，hook 掉 ClipboradManager 的相关方法，保证返回的 getPrimaryClip 的不为 null 即可。

于是看了几个点：
``` java
public @Nullable ClipData getPrimaryClip() {
    try {
        return mService.getPrimaryClip(mContext.getOpPackageName(), mContext.getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
``` 
这个 mService 的初始化为：
``` java
mService = IClipboard.Stub.asInterface(
                ServiceManager.getServiceOrThrow(Context.CLIPBOARD_SERVICE));
```                 

这么看，已经八成可以 hook 了，再看下我们自己能构造 ClipData 吗？
``` java
public ClipData(CharSequence label, String[] mimeTypes, Item item) {}
``` 
复制代码
恩，hook 的思路基本可行。

小张内心暗喜，多亏是遇到了我呀，还好我实力扎实。

这时候，资深研发老羊过来问了句，马上就要上线了，你这干啥呢？

小张滔滔不绝的描述了一下当前遇到了问题，和自己的解决思路，本以为老羊这次会拍拍自己的肩膀「还好是你遇到了呀」来表示对自己的认可。

这时候老羊说了句：

你也想想，假设三方库里面真有个致命的 bug，然后你没找到合适的 hook 点你怎么处理？想好了过来告诉我。

致命 bug，没找到合适的 hook 点？

模拟下代码：
``` java
public class Tools {

    public static void evilCode() {
        int a = 1 / 0;
    }

    public static String getClipBoardStr(Context context) {
        evilCode();
        ClipboardManager clipboardManager = (ClipboardManager) context.getSystemService(Context.CLIPBOARD_SERVICE);
        ClipData primaryClip = clipboardManager.getPrimaryClip();
        ClipData.Item itemAt = primaryClip.getItemAt(0);
        if (itemAt == null) {
            return "";
        }
        CharSequence text = itemAt.getText();
        if (text == null) {
            return "";
        }
        return text.toString();
    }
}
``` 

假设 getClipBoardStr 内部调用了一行 evilCode，执行到就 crash。

一眼望去这个 evilCode 方法，简单是简单，但是在三方库里面怎么解决呢？

小张百思不得其解，忽然灵光一闪：

是不是老羊想考察我的推动能力，让我没事别瞎 hook 人家代码，这种问题当然找三方库那边修复，然后给个新版本咯。

于是跑过去，告诉老羊，我想到了，这种问题，我们应该及时推动三方库那边解决，然后我们升级版本即可。

老羊听了后，恩，确实要找他们，但是如果是上线前遇到，推动肯定是来不及了，就是人家立马给你个新版本，直接升级风险也是比较大的。

然后老羊说道：

我看你对于反射找 hook 点已经比较熟悉了，其实还有一类 hook 更加好用，也更加稳定。

叫做字节码 hook。

怎么说？

我们的代码在打包过程中，会经过如下步骤：
``` 
.java -> .class -> dex -> apk
``` 

上面那个类的 evil 方法，从 class 文件的角度来看，其实都是字节码。

假设我们在编译过程中，这么做：
``` 
.java -> .class -> 拿到 Tools.class，修正里面的方法 evil 方法 -> dex -> apk
``` 

这个时机，其实构建过程中也给我们提供了，也就是传说的 Transform 阶段（这里不讨论 AGP 7 之后的变化，还是有对应时机的）。

小张又问，这个时机我知道，Tools.class 文件怎么修改呢？

老羊说，这个你去看看我的博客：

Android 进阶之路：ASM 修改字节码，这样学就对了！

不过话说回来，既然你会遇到这样的痛点，那么别的开发者肯定也会遇到。

这个时候应该怎么想？

小张：肯定有人造了好用的轮子。

老羊：恩，99% 的情况，轮子肯定都造好了，剩下 1%，那就是你的机会了。

轻量级 aop 框架 lancet 出现
饿了么，很早的时候就开源了一个框架，叫 lancet。

github.com/eleme/lance…

这个框架可以支持你，在不懂字节码的情况下，也能够完成对对应方法字节码的修改。

代入到我们刚才的思路：

.java -> .class -> lancet 拿到 Tools.class，修正里面的方法 evilCode 方法 -> dex -> apk

小张：怎么使用 lancet 来修改我们的 evilCode 方法呢？

引入框架
在项目的根目录添加：
``` grovvy
classpath 'me.ele:lancet-plugin:1.0.6'
``` 

在 module 的 build.gradle 添加依赖和 apply plugin：
``` grovvy
apply plugin: 'me.ele.lancet'

dependencies {
    implementation 'me.ele:lancet-base:1.0.6' // 最好查一下，用最新版本
}
``` 
开始使用
然后，我们做一件事情，把 Tools 里面的 evilCode 方法：

``` java    
public static void evilCode() {
    int a = 1 / 0;
}
``` 
里面的这个代码给去掉，让它变成空方法。

我们编写代码：
``` java
package com.imooc.blogdemo.blog04;

import me.ele.lancet.base.annotations.Insert;
import me.ele.lancet.base.annotations.TargetClass;

public class ToolsLancet {

    @TargetClass("com.imooc.blogdemo.blog04.Tools")
    @Insert("evilCode")
    public static void evilCode() {

    }

}
```
我们编写一个新的方法，保证其是个空方法，这样就完成让原有的 evilCode 中调用没有了。

其中：

TargetClass 注解：标识你要修改的类名；
Insert 注解：表示你要往 evilCode 这个方法里面注入下面的代码
下面的方法声明需要和原方法保持一致，如果有参数，参数也要保持一致（方法名、参数名不需要一致）
然后我们打包，看看背后发生了什么神奇的事情。

在打包完成后，我们反编译，看看 Tools.class
``` java
public class Tools {	
   //... 
    public static void evilCode() {
        Tools._lancet.com_imooc_blogdemo_blog04_ToolsLancet_evilCode();
    }

    private static void evilCode$___twin___() {
        int a = 1 / 0;
    }

    private static class _lancet {
        private _lancet() {
        }

        @TargetClass("com.imooc.blogdemo.blog04.Tools")
        @Insert("evilCode")
        static void com_imooc_blogdemo_blog04_ToolsLancet_evilCode() {
        }
    }
}
```
可以看到，原本的 evilCode 方法中的校验，被换成了一个生成的方法调用，而这个生成的方法和我们编写的非常类似，并且其为空方法。

而原来的 evilCode 逻辑，放在一个evilCode$___twin___()方法中，可惜这个方法没地方调用。

这样原有的 evilCode 逻辑就变成了一个空方法了。

我们可以大致梳理下原理：

lancet 会将我们注明需要修改的方法调用中转到一个临时方法中，这个临时方法你可以理解为和我们编写的方法逻辑基本保持一致。

然后将该方法的原逻辑也提取到一个新方法中，以备使用。

小张：确实很神奇，那这个原方法我们什么时候会使用呢？

老羊：很多时候，可能原有逻辑只是个概率很低的问题，比如发送请求，只有在超时等情况才发生错误，你不能粗暴的把人家逻辑移除了，你可能更想加个 try-catch 然后给个提示什么的。

这个时候你可以这么改：
``` java
package com.imooc.blogdemo.blog04;

import me.ele.lancet.base.Origin;
import me.ele.lancet.base.annotations.Insert;
import me.ele.lancet.base.annotations.TargetClass;

public class ToolsLancet {

    @TargetClass("com.imooc.blogdemo.blog04.Tools")
    @Insert("evilCode")
    public static void evilCode() {
        try {
            Origin.callVoid();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

```
我们再来看下反编译代码：
``` java
public class Tools {

    public static void evilCode() {
        Tools._lancet.com_imooc_blogdemo_blog04_ToolsLancet_evilCode();
    }

    private static void evilCode$___twin___() {
        int a = 1 / 0;
    }

    private static class _lancet {
        @TargetClass("com.imooc.blogdemo.blog04.Tools")
        @Insert("evilCode")
        static void com_imooc_blogdemo_blog04_ToolsLancet_evilCode() {
            try {
                Tools.evilCode$___twin___();
            } catch (Exception var1) {
                var1.printStackTrace();
            }

        }
    }
}
```
看到没，不出所料中转方法内部调用了原有方法，然后外层包了个 try-catch。

是不是很强大，而且相对于运行时反射相关的 hook 更加稳定，其实他就像你写的代码，只不过是直接改的 class。

小张：所以我早上遇到的剪切板崩溃问题，其实也可以利用 lancet 加一个 try-catch。

老羊：是的，挺会举一反三的，当然也从侧面反映出来字节码 hook 的强大之处，几乎不需要找什么 hook 点，只要你有方法，就能干涉。

另外，我给你介绍的都是最基础的 api，你下去好好看看 lancet 的其他用法。

小张：好嘞，又学到了。

新的问题又来了
过了几日，忽然项目又遇到一个问题：

用户未授权读取剪切板之前，不允许有读取剪切板的行为，否则认定为不合规。

小张听到这个任务，大脑快速运转：

这个读取剪切板行为的 API 是：
``` java
clipboardManager.getPrimaryClip();
``` 
搜索下项目中的调用，然后逐一修改。

先不说能不能搜索完整，这三方库里面肯定有，此外后续新增的代码如何控制呢？

另外之前学习 lancet，可以修改三方库代码，但是我也不能把包含 clipboardManager.getPrimaryClip 的方法全部列出来，一个个字节码修改？

还是解决不了后续新增，已经能保证全部搜出来呀。

最终心里嘀咕：别让我干，别让我干，八成是个坑。

这时候老羊来了句：这个简单，小张熟悉，他搞就行了。

小张：我...

重新思考一下，反正搜索出来，一一修改是不可能了。

那就从源头上解决：

系统肯定是通过 framework，system 进程那边去判断是否读取剪切板的。

那么我们只要把：
``` java
clipboardManager.getPrimaryClip
	IClipboard.getPrimaryClip(mContext.getOpPackageName(), mContext.getUserId());
```
内部的逻辑 hook 掉，换掉 IClipBoard 的实现，然后切到我们自己的逻辑即可。

懂了，这就是我之前想的系统服务的 hook 而已，难怪老羊安排给我，我给他说过这个。

于是乎... 我开启了一顿写模式...

此处代码略。（确实可以，不过非本文主要内容）

正完成了 Android 10.0 的测试，准备翻翻各个版本有没有源码修改，好适配适配，老羊走了过来。

说了句：这都两个小时过去了，你还没搞完？

小张：两个小时搞完？你来。

老羊：我让你自己看看 lancet 其他 api 你没看？

这个用 lancet 就是送分题你知道吗？看好：
``` java
public class ToolsLancet {

    // 模拟用户同意后的状态
    public static boolean isAuth = true;

    @TargetClass("android.content.ClipboardManager")
    @Proxy("getPrimaryClip")
    public ClipData getPrimaryClip() {
        if (isAuth) {
            return (ClipData) Origin.call();
        }
        // 这里也可以 return null,毕竟系统也 return null
        return new ClipData("未授权呢", new String[]{"text/plain"}, new ClipData.Item(""));
    }
}
```
小张：这个不行呀，android.content.ClipboardManager 类是系统的，不是我们写的，在打包阶段没有这个 class。

老羊：我当然知道，你仔细看，这次用的注解和上次有什么不同。

这次用的是:

@Proxy：意思就是代理，会代理 ClipboardManager. getPrimaryClip 到我们这个方法中来。
我们反编译看看：

原来的调用：
``` java
public static String getClipBoardStr(Context context) {
    ClipboardManager clipboardManager = (ClipboardManager) context.getSystemService(Context.CLIPBOARD_SERVICE);
    ClipData primaryClip = clipboardManager.getPrimaryClip();
    ClipData.Item itemAt = primaryClip.getItemAt(0);
    if (itemAt == null) {
        return "";
    }
    CharSequence text = itemAt.getText();
    if (text == null) {
        return "";
    }
    return text.toString();
}
```
反编译的调用：
``` java
public class Tools {
    public static String getClipBoardStr(Context context) {
        ClipboardManager clipboardManager = (ClipboardManager)context.getSystemService("clipboard");
        ClipData primaryClip = Tools._lancet.com_imooc_blogdemo_blog04_ToolsLancet_getPrimaryClip(clipboardManager);
        Item itemAt = primaryClip.getItemAt(0);
        if (itemAt == null) {
            return "";
        } else {
            CharSequence text = itemAt.getText();
            return text == null ? "" : text.toString();
        }
    }

    private static class _lancet {
    
        @TargetClass("android.content.ClipboardManager")
        @Proxy("getPrimaryClip")
        static ClipData com_imooc_blogdemo_blog04_ToolsLancet_getPrimaryClip(ClipboardManager var0) {
            return ToolsLancet.isAuth ? var0.getPrimaryClip() : new ClipData("未授权呢", new String[]{"text/plain"}, new Item(""));
        }
    }
}
``` 
看到没有，clipboardManager.getPrimaryClip()方法变成了Tools._lancet.com_imooc_blogdemo_blog04_ToolsLancet_getPrimaryClip，中转到了我们的 hook 实现。

这次明白了吧：

lancet 对于我们自己的类中方法，可以使用 @Insert 指令；
遇到系统的调用，我们可以针对调用函数使用 @Proxy 指令将其中转到中转函数；
好了，lancet 还有一些 api，你再下去好好看看。

完结
终于结束了，大家退出小张和老羊的对话场景。

其实字节码 hook 在 Android 开发过程中更为强大，比我们传统的找 Hook 点（单例，静态变量），然后反射的方式方便太多了，还有个最大的优势就是稳定。

当然 lancet hook 有个前提就是要明确知道方法调用，如果你想 hook 一个类的所有调用，那么写起来就有点费劲了，可能并不如动态代理那么方便。

好了，话说回来：

之前有个小伙去面试，被问到：

如何收敛三方库里面线程池的创建？

你有想法了吗？

