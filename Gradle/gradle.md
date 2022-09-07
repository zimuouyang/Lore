```groovy
task makeJar(type:org.gradle.api.tasks.bundling.Jar) {
    //指定生成的jar名
    baseName 'drisk'
    //从哪里打包class文件
    from('build/intermediates/classes/debug/com/ijm/drisk')
    //打包到jar后的目录结构
    into('com/ijm/drisk')
    //去掉不需要打包的目录和文件
    //exclude('ui', 'uuid')
    //过滤不需要打入jar包的文件（以ui，uuid，M开头的目录和文件）
    exclude('ui', 'uuid', 'M')
    //指定打包的class
    //include "com/xxx/drisk/*.class"
}
//dependsOn 表示在之前执行此方法
makeJar.dependsOn(build)
```

- baseName指定生成的jar名
- from 从哪里打包class文件
- into 打包后的目录结构
- exclude 过滤
- include  指定引入
- extension  扩展名
- destinationDir  目标目录

## gradlew cAT