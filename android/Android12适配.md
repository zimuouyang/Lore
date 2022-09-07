Android 12 快速适配要点

Android 12 需要更新适配点并不多，本篇主要介绍最常见的两个需要适配的点：[`android:exported`](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fmanifest%2Factivity-element%23exported) 和 [SplashScreen](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fui%2Fsplash-screen) 。

## 一、android:exported

**它主要是设置 `Activity` 是否可由其他应用的组件启动**， “`true`” 则表示可以，而 “`false`” 表示不可以。

若为 “`false`”，则 `Activity` 只能由同一应用的组件或使用同一用户 ID 的不同应用启动。

当然不止是 `Activity`， `Service` 和 `Receiver` 也会有 `exported` 的场景。

**一般情况下如果使用了 `intent-filter`，则不能将 `exported` 设置为 “`false`”**，不然在 `Activity` 被调用时系统会抛出 `ActivityNotFoundException` 异常。

相反如果没有 `intent-filter`，那就不应该把 `Activity` 的 `exported` 设置为`true` ，**这可能会在安全扫描时被定义为安全漏洞**。

而在 Android 12 的平台上，也就是使用 `targetSdkVersion 31` 时，那么你就需要注意：

**如果 `Activity` 、 `Service` 或 `Receiver` 使用 `intent-filter` ，并且未显式声明 `android:exported` 的值，App 将会无法安装。**

**详细更新可见： [《Android 12 自动适配 exported 深入解析避坑》](https://juejin.cn/post/7074018771161219103)**

这时候你可能会选择去 `AndroidManifest` 一个一个手动修改，但是如果你使用的 SDK 或者第三方库没有支持怎么办？或者你想要打出不同 target 平台的包？这时候下面这段 gradle 脚本可以给你省心：

#### com.android.tools.build:gradle:3.4.3 以下版本

经过测试支持的版本：

- **gradle:4.0.0 & gradle-6.1.1-all.zip**

```groovy
/**
 * 修改 Android 12 因为 exported 的构建问题
 */
android.applicationVariants.all { variant ->
    variant.outputs.all { output ->
        output.processResources.doFirst { pm ->
            String manifestPath = output.processResources.manifestFile
            def manifestFile = new File(manifestPath)
            def xml = new XmlParser(false, true).parse(manifestFile)
            def exportedTag = "android:exported"
            ///指定 space
            def androidSpace = new groovy.xml.Namespace('http://schemas.android.com/apk/res/android', 'android')

            def nodes = xml.application[0].'*'.findAll {
                //挑选要修改的节点，没有指定的 exported 的才需要增加
                (it.name() == 'activity' || it.name() == 'receiver' || it.name() == 'service') && it.attribute(androidSpace.exported) == null

            }
            ///添加 exported，默认 false
            nodes.each {
                def isMain = false
                it.each {
                    if (it.name() == "intent-filter") {
                        it.each {
                            if (it.name() == "action") {
                                if (it.attributes().get(androidSpace.name) == "android.intent.action.MAIN") {
                                    isMain = true
                                    println("......................MAIN FOUND......................")
                                }
                            }
                        }
                    }
                }
                it.attributes().put(exportedTag, "${isMain}")
            }

            PrintWriter pw = new PrintWriter(manifestFile)
            pw.write(groovy.xml.XmlUtil.serialize(xml))
            pw.close()
        }
    }

}
复制代码
```

#### com.android.tools.build:gradle:4.1.0 以上版本

经过测试支持的版本：

- **gradle:4.1.0 & gradle-6.5.1-all.zip**

```
/**
 * 修改 Android 12 因为 exported 的构建问题
 */

android.applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def processManifest = output.getProcessManifestProvider().get()
        processManifest.doLast { task ->
            def outputDir = task.multiApkManifestOutputDirectory
            File outputDirectory
            if (outputDir instanceof File) {
                outputDirectory = outputDir
            } else {
                outputDirectory = outputDir.get().asFile
            }
            File manifestOutFile = file("$outputDirectory/AndroidManifest.xml")
            println("----------- ${manifestOutFile} ----------- ")

            if (manifestOutFile.exists() && manifestOutFile.canRead() && manifestOutFile.canWrite()) {
                def manifestFile = manifestOutFile
                ///这里第二个参数是 false ，所以 namespace 是展开的，所以下面不能用 androidSpace，而是用 nameTag
                def xml = new XmlParser(false, false).parse(manifestFile)
                def exportedTag = "android:exported"
                def nameTag = "android:name"
                ///指定 space
                //def androidSpace = new groovy.xml.Namespace('http://schemas.android.com/apk/res/android', 'android')

                def nodes = xml.application[0].'*'.findAll {
                    //挑选要修改的节点，没有指定的 exported 的才需要增加
                    //如果 exportedTag 拿不到可以尝试 it.attribute(androidSpace.exported)
                    (it.name() == 'activity' || it.name() == 'receiver' || it.name() == 'service') && it.attribute(exportedTag) == null

                }
                ///添加 exported，默认 false
                nodes.each {
                    def isMain = false
                    it.each {
                        if (it.name() == "intent-filter") {
                            it.each {
                                if (it.name() == "action") {
                                    //如果 nameTag 拿不到可以尝试 it.attribute(androidSpace.name)
                                    if (it.attributes().get(nameTag) == "android.intent.action.MAIN") {
                                        isMain = true
                                        println("......................MAIN FOUND......................")
                                    }
                                }
                            }
                        }
                    }
                    it.attributes().put(exportedTag, "${isMain}")
                }

                PrintWriter pw = new PrintWriter(manifestFile)
                pw.write(groovy.xml.XmlUtil.serialize(xml))
                pw.close()

            }

        }
    }
}

复制代码
```

这段脚本你可以直接放到 `app/build.gradle` 下执行，也可以单独放到一个 gradle 文件之后 `apply` 引入，它的作用就是：

**在打包过程中检索所有没有设置 `exported` 的组件，给他们动态配置上 `exported`**。这里有个特殊需要注意的是，因为启动 `Activity` 默认就是需要被 Launcher 打开的，所以 `"android.intent.action.MAIN"` 需要 `exported` 设置为 `true` 。（**PS：应该是用 LAUNCHER 类别，这里故意用 MAIN**）

如果有需要，还可以自己增加判断设置了 `"intent-filter"` 的才配置 `exported` 。

⚠️ 更新，非常抱歉的是，**从 `gradle:4.2.0 & gradle-6.7.1-all.zip` 开始，TargetSDK 31 下脚本还是会有异常，因为在 `processDebugMainManifest` （有 Main） 的阶段，会直接扫描依赖库的  `AndroidManifest.xml` 然后直接报错，进不去 `processDebugManifest` 任务阶段接编译停止**。
所以拿不到 `mergerd_manifest` 下的文件，没办法进入 task ，也就是该脚本只能针对 `gradle:4.1.0` 安装 apk 到 Android12 的机器上， 有 `intent-filter` 但没有 exoprted 的适配问题，**不知道各位是否有什么好的建议吗？**

**详细更新可见： [《Android 12 自动适配 exported 深入解析避坑》](https://juejin.cn/post/7074018771161219103)**

## 二、SplashScreen

Android 12 新增加了 [`SplashScreen`](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroid%2Fwindow%2FSplashScreen) 的 API，它包括启动时的进入应用的动作、显示应用的图标画面，以及展示应用本身的过渡效果。

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ebe850346cf4f31a250cc6fe6601500~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp?)

它大概由如下 4 个部分组成，这里需要注意：

- 1 最好是矢量的可绘制对象，当然它可以是静态或动画形式。
- 2 是可选的，也就是图标的背景。
- 与自适应图标一样，前景的三分之一被遮盖 (3)。
- 4 就是窗口背景。

启动画面动画机制由进入动画和退出动画组成。

- 进入动画由系统视图到启动画面组成，这由系统控制且不可自定义。
- 退出动画由隐藏启动画面的动画运行组成。如果要[对其进行自定义](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fui%2Fsplash-screen%23customize-animation)，可以通过 [`SplashScreenView`](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroid%2Fwindow%2FSplashScreenView) 自定义。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36aee0fe63da4ce4915d41469d250826~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp?)

更详细的介绍这里就不展开了，有兴趣的可以自己看官方的资料： [developer.android.com/guide/topic…](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fui%2Fsplash-screen) ，这里主要介绍下如何适配和使用的问题。

**首先不管你的 TargetSDK 什么版本，当你运行到 Android 12 的手机上时，所有的 App 都会增加 `SplashScreen` 的功能**。

如果你什么都不做，那 App 的 Launcher 图标会变成 `SplashScreen` 界面的那个图标，而对应的原主题下 `windowBackground` 属性指定的颜色，就会成为 `SplashScreen` 界面的背景颜色。**这个启动效果在所有应用的冷启动和热启动期间会出现。**

其实不适配好像也没啥问题。

关于如何迁移和使用 `SplashScreen` 可以查阅官方详细文档： [developer.android.com/guide/topic…](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fui%2Fsplash-screen%2Fmigrate)

另外还可以参考 [《Jetpack 新成员 SplashScreen：打造全新的 App 启动画面》](https://juejin.cn/post/6997217571208445965) 这篇文章，文章详细介绍了如果使用官方的 `Jetpack` 库来让这个效果适配到更低的 Target 平台。

而正常情况下我们可以做的就是：

- 1、升级 `compileSdkVersion 31` 、 `targetSdkVersion 31` & `buildToolsVersion '31.0.0'`
- 2、 添加依赖 `implementation "androidx.core:core-splashscreen:1.0.0-alpha02"`
- 3、增加 `values-v31` 的目录
- 4、添加 `styles.xml` 对应的主题，例如：

```
<resources>
    <style name="LaunchTheme" parent="Theme.SplashScreen">
        <item name="windowSplashScreenBackground">@color/splashScreenBackground</item>
        <!--<item name="windowSplashScreenAnimatedIcon">@drawable/splash</item>-->
        <item name="windowSplashScreenAnimationDuration">500</item>
        <item name="postSplashScreenTheme">@style/AppTheme</item>
    </style>
</resources>
复制代码
```

- 5、给你的启动 `Activity` 添加这个主题，不同目录下使用不同主题来达到适配效果。

**PS: 我个人是一点都不喜欢这个玩意。**

## 三、其他

### 1、通知中心又又又变了

**Android 12 更改了可以完全自定义通知外观和行为，以前自定义通知能够使用整个通知区域并提供自己的布局和样式，现在它行为变了**。

使用 TargetSDK 为 31 的 App，包含自定义内容视图的通知将不再使用完整通知区域；而是使用系统标准模板。

此模板可确保自定义通知在所有状态下都与其他通知长得一模一样，例如在收起状态下的通知图标和展开功能，以及在展开状态下的通知图标、应用名称和收起功能，与 Notification.DecoratedCustomViewStyle 的行为几乎完全相同。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5df06b56400498997daf50ed968f7f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp?)

### 2、Android App Links 验证

Android App Links 是一种特殊类型的 DeepLink ，用于让 Web 直接在 Android 应用中打开相应对应 App 内容而无需用户选择应用。使用它需要执行以下步骤：

如何使用可查阅：[developer.android.com/training/ap…](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fapp-links%2Fverify-site-associations%23auto-verification)

使用 TargetSDK 为 31 的 App，系统对 [Android App Links](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fapp-links%2Fverify-site-associations%23auto-verification) 的验证方式进行了一些调整，这些调整会提升应用链接的可靠性。

如果你的 App 是依靠 Android App Links 验证在应用中打开网页链接，那么在为 Android App Links 验证添加 intent 过滤器时，请确保使用正确的格式，**尤其需要注意的是确保这些 `intent-filter` 包含 BROWSABLE 类别并支持 `https` 方案**。

### 3、安全和隐私设置

#### 3.1、大致位置

**使用 TargetSDK 为 31 的 App，用户可以请求应用只能访问大致位置信息**。

如果 App 请求 `ACCESS_COARSE_LOCATION` 但未请求 `ACCESS_FINE_LOCATION` 那么不会有任何影响。

TargetSDK 为 31 的 App 请求 `ACCESS_FINE_LOCATION` 运行时权限，还必须请求 `ACCESS_COARSE_LOCATION` 权限。当 App 同时请求这两个权限时，系统权限对话框将为用户提供以下新选项：

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac8beb47848143848b981d56a08e0e7e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp?)

#### 3.2、SameSite Cookie

Cookie 的 `SameSite` 属性决定了它是可以与任何请求一起发送，还是只能与同站点请求一起发送。

- 没有 `SameSite` 属性的 Cookie 被视为 `SameSite=Lax`。
- 带有 `SameSite=None` 的 Cookie 还必须指定 `Secure` 属性，这意味着它们需要安全的上下文，需要通过 HTTPS 发送。
- 站点的 HTTP 版本和 HTTPS 版本之间的链接现在被视为跨站点请求，因此除非将 Cookie 正确标记为 `SameSite=None; Secure`，否则 Cookie 不会被发送。

在 [WebView devtools](https://link.juejin.cn/?target=https%3A%2F%2Fchromium.googlesource.com%2Fchromium%2Fsrc%2F%2B%2FHEAD%2Fandroid_webview%2Fdocs%2Fdeveloper-ui.md%23launching-webview-devtools) 中 [切换界面标志 webview-enable-modern-cookie-same-site](https://link.juejin.cn/?target=https%3A%2F%2Fchromium.googlesource.com%2Fchromium%2Fsrc%2F%2B%2FHEAD%2Fandroid_webview%2Fdocs%2Fdeveloper-ui.md%23Flag-UI)，可以在测试设备上手动启用 SameSite 行为。

### 4、应用休眠

Android 12 在 Android 11（API 级别 30）中引入的[自动重置权限行为](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fpermissions%2Frequesting%23auto-reset-permissions-unused-apps) 的基础上进行了扩展。

如果 TargetSDK 为 31 的 App 用户几个月不打开，则系统会自动重置授予的所有权限并将 App 置于休眠状态。

更多可以查阅：[developer.android.com/topic/perfo…](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Ftopic%2Fperformance%2Fapp-hibernation)

### 5、PendingIntent

PendingIntent 如果没有指定 `FLAG_IMMUTABLE` 或 `FLAG_MUTABLE` 会直接报错。（TargetSDK 31 下）