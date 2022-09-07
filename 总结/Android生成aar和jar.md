## 生成Jar

- 在工程目录下的build.gradle加入代码

  ```groovy
  task againMakeJar(type: Copy) {
      def name = project.name //Library名称
      delete 'libs/' + name + '.jar' //删除之前的旧jar包
      from('build/intermediates/packaged-classes/release/') //从这个目录下取出默认jar包
      into('libs/') //将jar包输出到指定目录下
      include('classes.jar') //将文件包含进来
      rename('classes.jar', name + '.jar') //自定义jar包的名字
  }
  againMakeJar.dependsOn(build)
  ```

- 点击之后选择 library包下面的Tasks->other->againMakeJar方法 然后运行它, 就可以在libs下面找到相应的jar包

## 生成aar

- 选择library目录下的 Tasks->build->assembleRelease方法
- 等待运行完毕后去 buile->outputs->aar的文件夹下拷贝出来即可

