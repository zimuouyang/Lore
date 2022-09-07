# Gradle 中的 Task

## 一、Task 定义及配置

TaskContainer: 管理所有的 Task，如：增加、查找。

定义（创建）Task

```groovy
// 直接通过task函数去创建
task helloTask {
  println 'i am helloTask.'
}
 
// 通过TaskContainer去创建
this.tasks.create(name: 'helloTask2') {
  println 'i am helloTask 2.'
}
```

配置 Task

```groovy
// 给Task指定分组与描述
task helloTask(group: 'study', description: 'task study'){ // 语法糖
  ...
}
task helloTask {
  group 'study' // setGroup('study')
  description 'task study' // setDescription('task study')
  ...
}
```

Task 除了可以配置 group、description 外，还可以配置 name、type、dependsOn、overwrite、action。

结论：

- 给 Task 分组之后，该 task 会被放到指定组中，方便归类查找。（默认被分组到 other 中）
- 给 Task 添加描述，相当于给方法添加注释。

## 二、Task 的执行详情

Task 中 doFirst 与 doLast 的使用：

```groovy
task helloTask {
  println 'i am helloTask.'
  doFirst {
    println 'the task group is: ' + group
  }
  // doFirst、doLast可以定义多个
  doFirst {}
}
// 外部指定doFirst（会比在闭包内部指定的doFirst先执行）
helloTask.doFirst {
  println 'the task description is: ' + description
}
 
// 统计build执行时长
def startBuildTime, endBuildTime
this.afterEvaluate { Project project ->
  // 保证要找的task已经配置完毕
  def preBuildTask = project.tasks.getByName('preBuild') // 执行build任务时，第一个被执行的Task
  preBuildTask.doFirst {
    startBuildTime = System.currentTimeMillis()
  }
  def buildTask = project.tasks.getByName('build') // 执行build任务时，最后一个被执行的Task
  buildTask.doLast {
    endBuildTime = System.currentTimeMillis()
    println "the build time is: ${endBuildTime - startBuildTime}"
  }
}
```

结论：

- Task 闭包中直接编写的代码，会在配置阶段执行。可以通过 doFirst、doLast 块将代码逻辑放到执行阶段中执行。
- doFirst、doLast 可以指定多个。
- 外部指定的 doFirst、doLast 会比内部指定的先执行。
- doFirst、doLast 可以对 gradle 中提供的已有的 task 进行扩展。

## 三、Task 的执行顺序

task 执行顺序指定的三种方式：

1. dependsOn 强依赖方式
2. 通过 Task 输入输出指定（与第 1 种等效）
3. 通过 API 指定执行顺序

### 1、Task 的依赖

```groovy
// ============= dependsOn强依赖方式 =============
task taskX {
  doLast {
      println 'taskX'
  }
}
task taskY {
  doLast {
      println 'taskY'
  }
}
// 方式一：静态依赖
// task taskZ(dependsOn: taskY) // 依赖一个task
task taskZ(dependsOn: [taskX, taskY]) { // 依赖多个task，需要用数组[]表示
  doLast {
      println 'taskZ'
  }
}
// 方式二：静态依赖
taskZ.dependsOn(taskX, taskY)
// 方式三：动态依赖
task taskZ() {
  dependsOn this.tasks.findAll {    // 依赖所有以lib开头的task
    task -> return task.name.startsWith('lib')
  }
  doLast {
      println 'taskZ'
  }
}
```

其他：

- taskZ 依赖了 taskX 与 taskY，所以在执行 taskZ 时，会先执行 taskX、taskY。
- taskZ 依赖了 taskX 与 taskY，但 taskX 与 taskY 没有关系，它们的执行顺序是随机的。

### 2、Task 的输入输出

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy80MDUwNDQzLWJjYjExYWFhOGE5ZDQ4ZWEucG5nP2ltYWdlTW9ncjIvYXV0by1vcmllbnQvc3RyaXB8aW1hZ2VWaWV3Mi8yL3cvMTIwMC9mb3JtYXQvd2VicA?x-oss-process=image/format,png)

inputs 和 outputs 是 Task 的属性。
inputs 可以是任意数据类型对象，而 outputs 只能是文件（或文件夹）。
TaskA 的 outputs 可以作为 TaskB 的 inputs。

例子：writeTask 输入扩展属性，输出文件，readTask 输入 writeTask 的输出文件

```groovy
ext {
    versionCode = '1.0.0'
    versionName = '100'
    versionInfo = 'App的第1个版本，完成聊天功能'
    destFile = file('release.xml')
    if (destFile != null && !destFile.exists()) {
        destFile.createNewFile()
    }
}
 
task writeTask {
    inputs.property('versionCode', this.versionCode)
    inputs.property('versionName', this.versionName)
    inputs.property('versionInfo', this.versionInfo)
    outputs.file this.destFile
    doLast {
        def data = inputs.getProperties() // 返回一个map
        File file = outputs.getFiles().getSingleFile()
        // 将map转为实体对象
        def versionMsg = new VersionMsg(data)
        def sw = new StringWriter()
        def xmlBuilder = new MarkupBuilder(sw)
        if (file.text != null && file.text.size() <= 0) { // 文件中没有内容
            // 实际上，xmlBuilder将xml数据写入到sw中
            xmlBuilder.releases { // <releases>
                release { // <releases>的子节点<release>
                    versionCode(versionMsg.versionCode)
                    // <release>的子节点<versionCode>1.0.0<versionCode>
                    versionName(versionMsg.versionName)
                    versionInfo(versionMsg.versionInfo)
                }
            }
            // 将sw里的内容写到文件中
            file.withWriter { writer ->
                writer.append(sw.toString())
            }
        } else { // 已经有其它版本信息了
            xmlBuilder.release {
                versionCode(versionMsg.versionCode)
                versionName(versionMsg.versionName)
                versionInfo(versionMsg.versionInfo)
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
 
task readTask {
    inputs.file destFile
    doLast {
        def file = inputs.files.singleFile
        println file.text
    }
}
 
task taskTest(dependsOn: [writeTask, readTask]) {
    doLast {
        println '任务执行完毕'
    }
}
 
class VersionMsg {
    String versionCode
    String versionName
    String versionInfo
}
```

通过执行 *gradle taskTask* 之后，就可以在工程目录下看到 release.xml 文件了。

结论：

- 因为 writeTask 与 readTask 通过 inputs、outputs 产生了关联关系，所以，readTask 一定会在 writeTask 执行之后才执行。

### 3、Task API 指定顺序

task 指定执行顺序的 api 有：

- mustRunAfter : 强行指定在某个或某些 task 执行之后才执行。
- shouldRunAfter : 与 mustRunAfter 一样，但不强制。

```
task taskX {
    doLast {
        println 'taskX'
    }
}
task taskY {
    // shouldRunAfter taskX
    mustRunAfter taskX
    doLast {
        println 'taskY'
    }
}
task taskZ {
    mustRunAfter taskY
    doLast {
        println 'taskZ'
    }
}
```

通过执行 *gradle taskY taskZ taskX* 之后，可以看到终端还是按 taskX、taskY、taskZ 顺序执行的。

## 四、挂接到构建生命周期

例子：build 任务执行完成后，执行一个自定义 task

```
this.afterEvaluate { Project project ->
    def buildTask = project.tasks.getByName('build')
    if (buildTask == null) throw GradleException('the build task is not found')
    buildTask.doLast {
        taskZ.execute()
    }
}
```

例子：Tinker 将自定义的 manifestTask 插入到了 gradle 脚本中 processManifest 与 processResources 这两个任务之间

```
TinkerManifestTask manifestTask = project.tasks.create("tinkerProcess${variantName}Manifest", TinkerManifestTask)
...
manifestTask.mustRunAfter variantOutput.processManifest
variantOutput.processResources.dependsOn manifestTask
```

## 五、Task 类型

[Gradle DSL Version 5.1](https://docs.gradle.org/current/dsl/)

[Copy - Gradle DSL Version 5.1](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Copy.html)