Android 自定义 Gradle 插件的 3 种方式 - 简书

## 前言

Gradle 插件在 Android 中的应用很广泛，很多字节码插桩方案就用到了这方面的知识，Android 官方提供了很多可用的插件，比如`apply plugin: 'com.android.application'`: 它表示生成一个 apk 应用的插件；`apply plugin: 'com.android.library'`: 它表示生成 AAR 包。
本文只是为入门 Gradle 插件提供一些思路与实践方案，不深入解析 Gradle 工作原理和 Task 相关的内容。
Gradle 插件官方文档地址：
[https://docs.gradle.org/current/userguide/custom_plugins.html](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fuserguide%2Fcustom_plugins.html)

## Gradle Plugins 简介

Gradle 插件打包了可重用的构建逻辑，可以在不同的项目中使用。Gradle 提供了几种方式来让你实现自定义插件，这样你可以重用你的构建逻辑，甚至提供给他人使用。

可以使用多种语言来实现 Gradle 插件，其实只要最终被编译为 JVM 字节码的都可以，常用的有`Groovy`、`Java`、`Kotlin`。通常，使用 Java 或 Kotlin（静态类型）实现的插件比使用 Groovy 实施的插件性能更好。

## 打包插件的 3 种方式

- Build script：在`build.gradle`构建脚本中直接使用，只能在本文件内使用；
- buildSrc project：新建一个名为`buildSrc`的 Module 使用，只能在本项目中使用；
- Standalone project：在独立的 Module 中使用，可以发布到本地或者远程仓库供其他项目使用。

## Build script

直接在构建脚本中编写插件代码，并应用插件。比如说在`app`的`build.gradle`中加入如下代码：

![img](http://upload-images.jianshu.io/upload_images/4134622-0667aff259f5a5a8.png)

Build script.png

按照官方文档的做法，`build.gradle`内引入上面的代码，会发现`Plugin`和`Project`2 个类是无法被引入的。而且这种方案有个弊端，只能在构建脚本文件内部使用，这样就没办法提供给其他`module`或者`project`使用了。**这种方案基本不会在真实项目中使用**

注意：对于`Gradle`而言，每一个`Module`都是一个项目。先知道这个概念，后面会解释。

## buildSrc 项目

1、创建好项目之后，新建一个名称为`buildSrc`的 Module，项目类型任意，只保留`build.gradle`文件和`src/main`目录，其余文件全部删掉。**注意：名字一定要是`buildSrc`，否则应用插件的时候会找不到插件。**修改后的目录如下：

![img](http://upload-images.jianshu.io/upload_images/4134622-5dfac4f9dc6d08c5.png)

buildSrc.png

**踩过的坑：**创建`buildSrc`这个 Module 的时候，如果选择了`Android Library`类型会有`Plugin with id 'com.android.library' not found.`的异常，这是因为 buildSrc 是 Android 的保留名称，只能作为 plugin 插件使用，后面修改 buildSrc 的`build.gradle`文件后就不报错了。如果选择`Java Library`类型，好像就没有这个异常，而且这个类型的文件少一些，为了方便，建议大家选择`Java Library`类型。

2、修改 Gradle 文件内容：

```groovy
apply plugin: 'groovy'  //必须
apply plugin: 'maven'

dependencies {
    implementation gradleApi() //必须
    implementation localGroovy() //必须
    //如果要使用android的API，需要引用这个，实现Transform的时候会用到
    //implementation 'com.android.tools.build:gradle:3.3.0'
}

repositories {
    //google()
    jcenter()
    mavenCentral() //必须
}
```

注意：如果引入了`com.android.tools.build:gradle:3.3.0`，需要加入`google()`仓库，我试过只引入 `jcenter()`和 `mavenCentral()`仓库中，会提示找不到。

3、在`main`下新建`groovy`目录，在`groovy`目录下创建`包名`目录，在`包名`目录下新建一个`groovy`文件，并且实现`org.gradle.api.Plugin`接口，注意文件名需要以`.groovy`结尾。

4、在`main`下新建`resources`目录，在`resources`目录下新建`META-INF`目录，再在`META-INF`下新建`gradle-plugins`目录, 在`gradle-plugins`目录下新建`properties`文件，比如`com.zx.plugin.properties`, 注意：这个文件命名是没有要求的，但是要以`.properties`结尾。

buildSrc 项目目录如下：

![img](http://upload-images.jianshu.io/upload_images/4134622-939776986b91b59c.png)

项目结构

`CusPlugin.groovy`源码如下：
注意：这里是 groovy 语言写的，当然也可以用 java、kotlin 写，他们都是基于 JVM 的。`Plugin`和`Project`是 Gradle 的 API，所以需要先在脚本文件中配置好了再写插件实现类，否则是找不到这 2 个类的。

```groovy
package com.zx.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project


class CusPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        println("this is CusPlugin")
    }
}
```

这里没有去定义一系列 Task 任务，只是简单的打印 Log。在实际应用中，会定义一些任务去执行，这个后面的文章会讲。

`META-INF/gradle-plugins/com.zx.plugin.properties`文件的内容如下：

```
implementation-class=com.zx.plugin.CusPlugin
```

它的作用是：申明 Gradle 插件的具体实现类

5、在要使用插件的 Module 中应用，比如在`app`的`build.gradle`中，引用插件如下：

```groovy
apply plugin: 'com.android.application'
//引用自定义插件
apply plugin: 'com.zx.plugin'
```

**注意这里引用的插件名称就是`properties`文件的名称**。接下来如果编译正常的话，就会看见插件实现类的`apply()`方法中打印的日志。

![img](http://upload-images.jianshu.io/upload_images/4134622-cebb7fcab40c5def.png)

运行结果

小结：

- module 名称只能为`buildSrc`；
- buildSrc project 下的插件是自动加载。

## 独立的项目使用

这种方案的 Module 名称可以自定义，可以发布到本地或者远程仓库 (jcenter、maven 等）中，这样就可以供其他项目使用。

1、新建一个 Module，项目类型任意，名字任意，也是只保留`build.gradle`文件和`src/main`目录，其余文件全部删掉。

2、修改 Gradle 文件内容：

```groovy
apply plugin: 'groovy'  //必须
apply plugin: 'maven'  //要想发布到Maven，此插件必须使用


dependencies {
    implementation gradleApi() //必须
    implementation localGroovy() //必须
}
repositories {
    mavenCentral() //必须
}


def group='com.zx.cus_plugin' //组
def version='1.0.0' //版本
def artifactId='myGradlePlugin' //唯一标示


//将插件打包上传到本地maven仓库
uploadArchives {
    repositories {
        mavenDeployer {
            pom.groupId = group
            pom.artifactId = artifactId
            pom.version = version
            //指定本地maven的路径，在项目根目录下
            repository(url: uri('../repos'))
        }
    }
}
```

相比`buildSrc`方案，增加了`Maven`的支持和`uploadArchives`这样一个 Task，这个 Task 的作用是为了将插件打包上传到本地 maven 仓库。注意打包文件目录是`../repos`，它表示的是项目根目录下，这里用了 2 个`.`，1 个`.`表示当前 module 根目录，2 个`.`表示 project 的根目录。

3、`src/main`目录下的插件实现类和`properties`文件与`buildSrc`方案是一致的。注意 `properties` 的名称就是Plugin 的id

4、在终端中执行 gradle uploadArchives 指令，或者展开 AS 右侧的 Gradle，找到对应 module`uploadArchives`Task，就可以将插件部署到项目根目录的`repos`目录下。

![img](http://upload-images.jianshu.io/upload_images/4134622-b71a21cc192f7df7.png)

uploadArchives

插件部署到本地后的目录如下：
这个`repos`就是你本地的 Maven 仓库，`com/zx/cus_plugin`是脚本中的`group`指定的，`myGradlePlugin`表示模块名称，是一个唯一标示，`1.0.0`由`version`指定

![img](http://upload-images.jianshu.io/upload_images/4134622-801c37fe1cc9f482.png)

repos 目录

5、引用插件
**在`buildSrc`中，系统自动帮开发者自定义的插件提供了引用支持，但完全自定义 Module 的插件中，开发者就需要自己来添加自定义插件的引用支持。**

在 project 的`build.gradle`文件中，添加如下脚本：

```groovy
buildscript {
    repositories {
        google()
        jcenter()
        maven {
            url uri('./repos') //指定本地maven的路径，在项目根目录下
        }
    }
    dependencies {
        //classpath 'com.android.tools.build:gradle:3.3.0'
        classpath 'com.zx.cus_plugin:myGradlePlugin:1.0.0'
    }
}
```

注意：

- 这里的 uri 只用了 1 个`.`，因为 project 的`build.gradle`文件已经在项目根目录下了
- `classpath`指定的路径格式如下：
  这 3 个参数是在`build.gradle`脚本文件中申明的

```
classpath '[groupId]:[artifactId]:[version]' 
```

配置完毕后，在需要使用的 Module 引用插件，例如：

```
apply plugin: 'com.android.application'
//引用buildSrc插件，properties文件名称的方式
//apply plugin: 'com.zx.plugin'
//引用完全自定义插件，properties文件名称的方式
apply plugin: 'com.zx.cus_plugin'
```

小结：

- 方案 3 用完全自定义 Module 实现自定义插件，这里是打包上传插件到本地 Maven，当然你也可以发布到远程仓库，参考：[gradle 插件上传 Jcenter 与自建 Maven 私服](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fpf_1308108803%2Farticle%2Fdetails%2F78119591)
- 上传本地 Maven 需要注意脚本文件中`uploadArchives`配置，以及引用插件时地址的配置（我在这里踩过坑），一般放在 project 根目录下就可以
- 如果重新修改了插件 代码，需要重新部署`uploadArchives`才能在别的地方引用插件

## 总结

虽然官方提供了 3 种方案，但实际开发中只会用到后 2 种，有一个比较方便的做法，开发调试的时候用`buildSrc` project 模式，等后面需要把插件共享出去等时候，再把它改为第三种方案即可。



## 实现扩展属性

1. ​	创建一个实体类，声明成员变量，用于接收gradle中配置的参数。（可以理解为就是javaBean，不过要注意，该文件后缀是.groovy，不是.java）

   1. ```groovy
      class ReleaseInfoExtension {
          String versionCode
          String versionName
          String versionInfo
          String fileName
       
          ReleaseInfoExtension() {}
       
          @Override
          String toString() {
              return "versionCode = ${versionCode} , versionName = ${versionName} ," +
                      " versionInfo = ${versionInfo} , fileName = ${fileName}"
          }
      }
      ```

2. 在自定义插件中，对当前project进行扩展。

   ```groovy
   class GradleStudyPlugin implements Plugin<Project> {
    
       /**
        * 插件引入时要执行的方法
        * @param project 引入当前插件的project
        */
       @Override
       void apply(Project project) {
           // 这样就可以在gradle脚本中，通过releaseInfo闭包来完成ReleaseInfoExtension的初始化。
           project.extensions.create("releaseInfo", ReleaseInfoExtension)
       }
   }
   ```

3. 打开在app工程的build.gradle，通过扩展key值命名闭包的方式，就可以配置指定参数了。

   ```groovy
   apply plugin: 'com.lqr.gradle.study'
    
   releaseInfo {
       versionCode = '1.0.0'
       versionName = '100'
       versionInfo = '第一个app信息'
       fileName = 'release.xml'
   }
   ```

4. 接收参数，如：

   ```groovy
   def versionCodeMsg = project.extensions.releaseInfo.versionCode
   ```

## 创建扩展Task

​	扩展Task也很简单，继承DefaultTask，编写TaskAction注解方法，下面以 “把app版本信息写入到xml文件中”的task 为例，注释很详细，不多赘述：

```groovy
import groovy.xml.MarkupBuilder
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
 
class ReleaseInfoTask extends DefaultTask {
 
    ReleaseInfoTask() {
        group 'lqr' // 指定分组
        description 'update the release info' // 添加说明信息
    }
 
    /**
     * 使用TaskAction注解，可以让方法在gradle的执行阶段去执行。
     * doFirst其实就是在外部为@TaskAction的最前面添加执行逻辑。
     * 而doLast则是在外部为@TaskAction的最后面添加执行逻辑。
     */
    @TaskAction
    void doAction() {
        updateInfo()
    }
 
    private void updateInfo() {
        // 获取gradle脚本中配置的参数
        def versionCodeMsg = project.extensions.releaseInfo.versionCode
        def versionNameMsg = project.extensions.releaseInfo.versionName
        def versionInfoMsg = project.extensions.releaseInfo.versionInfo
        def fileName = project.extensions.releaseInfo.fileName
        // 创建xml文件
        def file = project.file(fileName)
        if (file != null && !file.exists()) {
            file.createNewFile()
        }
        // 创建写入xml数据所需要的类。
        def sw = new StringWriter();
        def xmlBuilder = new MarkupBuilder(sw)
        // 若xml文件中没有内容，就多创建一个realease节点，并写入xml数据
        if (file.text != null && file.text.size() <= 0) {
            xmlBuilder.releases {
                release {
                    versionCode(versionCodeMsg)
                    versionName(versionNameMsg)
                    versionInfo(versionInfoMsg)
                }
            }
            file.withWriter { writer ->
                writer.append(sw.toString())
            }
        } else { // 若xml文件中已经有内容，则在原来的内容上追加。
            xmlBuilder.release {
                versionCode(versionCodeMsg)
                versionName(versionNameMsg)
                versionInfo(versionInfoMsg)
            }
            def lines = file.readLines()
            def lengths = lines.size() - 1
            file.withWriter { writer ->
                lines.eachWithIndex { String line, int index ->
                    if (index != lengths) {
                        writer.append(line + '\r\n')
                    } else if (index == lengths) {
                        writer.append(sw.toString() + '\r\n')
                        writer.append(line + '\r\n')
                    }
                }
            }
        }
    }
}
```

与创建扩展属性一样，扩展Task也需要在project中创建注入。

```groovy
/**
 * 自定义Gradle插件
 */
class GradleStudyPlugin implements Plugin<Project> {
 
    /**
     * 插件引入时要执行的方法
     * @param project 引入当前插件的project
     */
    @Override
    void apply(Project project) {
        // 创建扩展属性
        // 这样就可以在gradle脚本中，通过releaseInfo闭包来完成ReleaseInfoExtension的初始化。
        project.extensions.create("releaseInfo", ReleaseInfoExtension)
        // 创建Task
        project.tasks.create("updateReleaseInfo", ReleaseInfoTask)
    }
}
```

