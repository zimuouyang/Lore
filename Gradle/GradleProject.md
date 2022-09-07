# 初识 [Gradle]

## 一、Gradle 的基本概念

一个开源的项目自动化构建工具，建立在 [Apache](https://so.csdn.net/so/search?q=Apache&spm=1001.2101.3001.7020) Ant 和 Apache Maven 概念的基础上，并引入了基于 Groovy 的特定领域语言（DSL）, 而不再使用 XML 形式管理构建脚本。同时，gradle 还是一个编程框架，可以让开发者使用编程的思想来实现应用构建。gradle 的组成：

1. groovy 核心语法
2. build script block
3. gradle api

## 二、Gradle 的执行流程

| 生命周期                 | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| Initialzation 初始化阶段 | 解析整个工程中所有 project（读取 setting.gradle 文件），构建所有的 project 对应的 Project 对象 |
| Configuration 配置阶段   | 解析所有的 project 对象中的 task，构建好所有 task 的拓扑图   |
| Execution 执行阶段       | 执行具体的 task 及其依赖的 task                              |

生命周期监听：

```
// 配置阶段开始前的监听回调（即：在Initialzation与Configuration之间）
this.beforeEvaluate {}
// 配置阶段完成后的监听回调（即：在Configuration与Execution之间）
this.afterEvaluate {}
// gradle执行完毕后的回调监听（即：在Execution之后）
this.gradle.buildFinished {}
 
// 与 this.beforeEvaluate {} 一样
this.gradle.beforeProject {}
// 与 this.afterEvaluate {} 一样
this.gradle.afterProject {}
```

# Gradle 中的 Project

## 一、Idea 与 Gradle 对于 project 概念的区别

在 Idea 中，一个项目就是 Project，一个 Project 下可以包含多个模块（Module），一个 Module 下，又可以有多个 Module，其树状结构如下：

虽然可以在 Module 中继续创建子 Module，但一般情况下，我们不会这么做，最多就两层。

```
+Project
--+Module
----+Module
----+Module
--+Module
--+Module
```

而对于 Gradle 而言，Idea 中的无论是 Project 还是 Module，都是 project，故树状结构如下：

每个 project 下，都一定会有一个 build.gradle

```
+project        // rootProject
--+project      // subProject
----+project
----+project
--+project
--+project
```

## 二、project 相关 [api]

| api                                            | 作用                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ |
| getAllprojects()                               | 获取工程中所有的 project（包括根 project 与子 project）      |
| getSubProjects()                               | 获取当前 project 下，所有的子 project（在不同的 project 下调用，结果会不一样，可能返回 null） |
| getParent()                                    | 获取当前 project 的父 project（若在 rooProject 的 build.gradle 调用，则返回 null） |
| getRootProject()                               | 获取项目的根 project（一定不会为 null）                      |
| project(String path, Closure configureClosure) | 根据 path 找到 project，通过闭包进行配置（闭包的参数是 path 对应的 Project 对象） |
| allprojects(Closure configureClosure)          | 配置当前 project 和其子 project 的所有 project               |
| subprojects(Closure configureClosure)          | 配置子 project 的所有 project（不包含当前 project）          |

```
// rootProject build.gradle下配置：
// 1、Project project(String path, Closure configureClosure);
project('app') { Project project ->       // 一个参数时，可以省略不写，这里只是为了明确参数的类型
  apply plugin : 'com.android.application'
  group 'com.lqr'
  version '1.0.0-release'
  dependencies {}
  android {}
}
 
// 2、allprojects(Closure configureClosure)
allprojects {
  group 'com.lqr'
  version '1.0.0-release'
}
 
// 3、subprojects(Closure configureClosure)
subprojects { Project project -> 
  if(project.plugins.hasPlugin('com.android.library')){
    apply from: '../publishToMaven.gradle'
  }
}
```

## 三、属性相关 api

### 1、在 gradle 脚本文件中使用 ext 块扩展属性

父 project 中通过 ext 块定义的属性，子 project 可以直接访问使用

```
// rootProject : build.gradle
// 定义扩展属性
ext {
  compileSdkVersion = 25
  libAndroidDesign = 'com.android.support:design:25.0.0'
}
 
// app : build.gradle
android {
  compileSdkVersion = this.compileSdkVersion // 父project中的属性，子project可以直接访问使用
  ...
}
dependencies {
  compile this.libAndroidDesign // 也可以使用：this.rootproject.libAndroidDesign
  ...
}
 
```

### 2、在 gradle.properties 文件中扩展属性

hasProperty('xxx')：判断是否有在 gradle.properties 文件定义 xxx 属性。
在 gradle.properties 中定义的属性，可以直接访问，但得到的类型为 Object，一般需要通过 toXXX() 方法转型。

```
// gradle.properties
// 定义扩展属性
isLoadTest=true
mCompileSdkVersion=25
 
// setting.gradle
// 判断是否需要引入Test这个Module
if(hasProperty('isLoadTest') ? isLoadTest.toBoolean() : false) {
  include ':Test'
}
 
// app : build.gradle
android {
  compileSdkVersion = mCompileSdkVersion.toInteger()
  ...
}
```

## 四、文件相关 api

| api                                                | 作用                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| getRootDir()                                       | 获取 rootProject 目录                                        |
| getBuildDir()                                      | 获取当前 project 的 build 目录（每个 project 都有自己的 build 目录） |
| getProjectDir()                                    | 获取当前 project 目录                                        |
| File file(Object path)                             | 定位一个文件，相对于当前 project 开始查找                    |
| ConfigurableFileCollection files(Object... paths)  | 定位多个文件，与 file 类似                                   |
| copy(Closure closure)                              | 拷贝文件                                                     |
| fileTree(Object baseDir, Closure configureClosure) | 定位一个文件树（目录 + 文件），可对文件树进行遍历            |

```
// 打印common.gradle文件内容
println getContent('common.gradle')
def getContent(String path){
  try{
    def file = file(path)
    return file.text
  }catch(GradleException e){
    println 'file not found..'
  }
  return null
}
 
// 拷贝文件、文件夹
copy {
  from file('build/outputs/apk/')
  into getRootProject().getBuildDir().path + '/apk/'
  exclude {} // 排除文件
  rename {} // 文件重命名
}
 
// 对文件树进行遍历并拷贝
fileTree('build/outputs/apk/') { FileTree fileTree ->
    fileTree.visit { FileTreeElement element ->
        println 'the file name is: '+element.file.name
        copy {
            from element.file
            into getRootProject().getBuildDir().path + '/test/'
        }
    }
}
```



## 五、依赖相关 api

配置工程仓库及 gradle 插件依赖

```
// rootProject : build.gradle
buildscript { ScriptHandler scriptHandler ->
    // 配置工程仓库地址
    scriptHandler.repositories { RepositoryHandler repositoryHandler ->
        repositoryHandler.jcenter()
        repositoryHandler.mavenCentral()
        repositoryHandler.mavenLocal()
        repositoryHandler.ivy {}
        repositoryHandler.maven { MavenArtifactRepository mavenArtifactRepository ->
            mavenArtifactRepository.name 'personal'
            mavenArtifactRepository.url 'http://localhost:8081/nexus/repositories/'
            mavenArtifactRepository.credentials {
                username = 'admin'
                password = 'admin123'
            }
        }
    }
    // 配置工程的"插件"（编写gradle脚本使用的第三方库）依赖地址
    scriptHandler.dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath 'com.tencent.tinker-patch-gradle-plugin:1.7.7'
    }
}
 
// ============ 上述脚本简化后 ============
 
buildscript {
    // 配置工程仓库地址
    repositories {
        jcenter()
        mavenCentral()
        mavenLocal()
        ivy {}
        maven {
            name 'personal'
            url 'http://localhost:8081/nexus/repositories/'
            credentials {
                username = 'admin'
                password = 'admin123'
            }
        }
    }
    // 配置工程的"插件"（编写gradle脚本使用的第三方库）依赖地址
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath 'com.tencent.tinker-patch-gradle-plugin:1.7.7'
    }
}
```

配置应用程序第三方库依赖

compile: 编译依赖包并将依赖包中的类打包进 apk。
provided: 只提供编译支持，但打包时依赖包中的类不会写入 apk。

```
// app : build.gradle
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar']) // 依赖文件树
    // compile file() // 依赖单个文件
    // compile files() // 依赖多个文件
    compile 'com.android.support:appcompat-v7:26.1.0' // 依赖仓库中的第三方库（即：远程库）
    compile project('mySDK') { // 依赖工程下其他Module（即：源码库工程）
      exclude module: 'support-v4' // 排除依赖：排除指定module
      exclude group: 'com.android.support' // 排除依赖：排除指定group下所有的module
      transitive false // 禁止传递依赖，默认值为false
    }
  
    // 栈内编译
    provided('com.tencent.tinker:tinker-android-anno:1.9.1')
}
```

provided 的使用场景：

1. 依赖包只在编译期起作用。（如：tinker 的 tinker-android-anno 只用于在编译期生成 Application，并不需要把该库中类打包进 apk，这样可以减小 apk 包体积）
2. 被依赖的工程中已经有了相同版本的第三方库，为了避免重复引用，可以使用 provided。

## 六、外部命令 api

```
// copyApk任务：用于将app工程生成出来apk目录及文件拷贝到本机下载目录
task('copyApk') {
    doLast {
        // gradle的执行阶段去执行
        def sourcePath = this.buildDir.path + '/outputs/apk'
        def destinationPath = '/Users/lqr/Downloads'
        def command = "mv -f ${sourcePath} ${destinationPath}"
        // exec块代码基本是固定的
        exec {
            try {
                executable 'bash'
                args '-c', command
                println 'the command is executed success.'
            }catch (GradleException e){
                println 'the command is executed failed.'
            }
        }
    }
}
```