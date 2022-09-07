## 1. Image

该关键字指定一个任务（job）所使用的docker镜像，例如`image: python:latest`使用Python的最新镜像。

#### 镜像下载策略 (gitlab runner config.toml 中配置)

- never： 当使用这个策略，会禁止Gitlab Runner从Docker hub或者其他地方下拉镜像，只能使用自己手动下拉的镜像
- if-not-present： 当使用这个策略，Runner会先检测本地是否有镜像，有的话使用该镜像，如果没有再去下拉。这个策略如果再配合定期删除镜像，就能达到比较好的效果。
- always： 这个是gitlab-ci默认使用的策略，即每一次都是重新下拉镜像，导致的结果就是比较耗时间



## 2. services

​	该关键字指向其他Docker镜像，这些镜像会与image关键字指定的镜像绑定（link）。比如可以绑定一个mysql服务，存储单元测试所需要的假数据。

```yaml
default_job:
  image: python:3.6

  services:
    - mysql:5.7

  before_script:
    - apt-get install mysql-client

```

## 3. before_script和after_script

​	before_script关键字定义了一组在每个任务开始之前需要执行的命令，after_script则相反

## 4.stages

​	前面提到，我们可以定义一系列任务（job）去执行我们想要的指令，但怎样指定先后顺序呢，这就需要用到stages关键字了。

​	stages关键字有两个特性：

​	1.如果两个任务对应的stage名相同，则这两个任务会并行运行
​	2.下一个stage关联的任务会等待上一个stage执行成功后才继续运行，失败则不运行

```yaml
image: python:3.6

stages:
  - build
  - test
  - deploy
# build-job1和build-job2会并行执行
build-job1:
  stage: build
  script:
    - echo $PWD

build-job2:
  stage: build
  script:
    - echo $PWD

# 这个任务会在build-job1和build-job2执行成功后再运行
test-job:
  stage: test
  script:
    - echo $PWD

# 注意定义的stage都需要用到
deploy-job:
  stage: deploy
  script:
    - echo $PWD
```

**注：如果想要控制某一个stage在最开始，或者最后执行，可以使用`.pre` 和 `.post` 关键字**

## 5.only / except

​	通过stages关键字可以控制任务执行的先后顺序，而通过`only/except`关键字控制的是任务的触发条件。

​	only/except关键字包含了一些子关键字（或者你可以理解为策略），当符合定义的策略时才会触发Pipelines的执行，except则相反。

​	only关键字的默认策略是['branches', 'tags']，即你提交了一个分支或者打了标签，就会触发，

- ​	当提交的分支为master才会触发Pipelines

```yaml
image: python:3.6

# 第一种方式
job1:
  only:
    - master
  script:
    - echo $PWD

# 第二种方式
job2:
  only:
    refs:
      - master
  script:
    - echo $PWD

# 第三种方式
job3:
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "master"
  script:
    - echo $PWD
```

- 当提交的分支为master或者打了tags触发Pipelines

  ```yaml
  image: python:3.6
  
  # 举这个例子是想说明only关键字之间的关系是”或“关系
  job1:
    only:
      - master
      - tags
    script:
      - echo $PWD
  ```

- 当使用Web界面的`Run Pipelines`时触发

  ```yaml
  image: python:3.6
  
  # 举这个例子是想说明only关键字之间的关系是”或“关系
  job1:
    only:
      - web
    script:
      - echo $PWD
  ```

## 6.tags

  该关键字指定了使用哪个Runner（哪个机器）去执行我们的任务，注意与上文only关键字的tags进行区分。

```yaml
image: python:3.6

# 举这个例子是想说明only关键字之间的关系是”或“关系
job1:
  only:
    - dev
  tags:
    - machine1
  script:
    - echo $PWD
```

## 7.when

​	前面说过，stages关键字可以控制每个任务的执行顺序，且后一个stage会等待前一个stage执行成功后才会执行，那如果我们想要达到前一个stage失败了，后面的stage仍然能够执行的效果呢？

这时候when关键字就可以发挥作用了，它一共有五个值：

- on_success：只有前面stages的所有工作成功时才执行，这是默认值。
- on_failure：当前面stages中任意一个jobs失败后执行
- always：无论前面stages中jobs状态如何都执行
- manual：手动执行
- delayed：延迟执行

```yaml
# 官方示例
stages:
  - build
  - cleanup_build
  - test
  - deploy
  - cleanup

build_job:
  stage: build
  script:
    - make build

# 如果build_job任务失败，则会触发该任务执行
cleanup_build_job:
  stage: cleanup_build
  script:
    - cleanup build when failed
  when: on_failure

test_job:
  stage: test
  script:
    - make test

deploy_job:
  stage: deploy
  script:
    - make deploy
  when: manual

# 总是执行
cleanup_job:
  stage: cleanup
  script:
    - cleanup after jobs
  when: always

```

## 8.cache

​	该关键字指定了需要缓存的文件夹或者文件，目的是为了加快执行速度。

值得注意的是，cache不能缓存工作路径以外的文件。比如你的用户是yangan，项目是helloworld，默认的工作路径就是/build/yangan/helloworld, 如果你想缓存/root下的某个文件，就会出现“找不到文件”的错误。

另外，cache并不能保证每次都能命中，而且如果缓存的文件较大，有时候反而会适得其反，导致速度变慢。

```yaml
image: python:3.6

cache:
  # 注意是相对路径
  paths:
    - cache_dir/


job1:
  script: echo $PWD
  # 针对单个任务
  cache:
    - binaries/
```

## 9.artifacts

​	类似cache关键字，也可以缓存文件或者文件夹，不同的是，这些文件可以在Gitlab的UI界面中下载，一般可用来存储Android打包生成的apk。

```yaml

image: python:3.6

job1:
  artifacts:
    paths:
      - binaries/
    # 设置过期时间
    expire_in: 1 week
```

## 10.变量

​	Gitlab-CI的变量有几种，分别为预先定义的环境变量，比如`CI_COMMIT_SHA`, `CI_COMMIT_MESSAGE`; 通过Web界面Settings/CI-CD设置的变量，以及自己在`.gitlab-ci.yml`定义的变量

```yaml
variables:
  TEST: "HELLO WORLD"
  
 script:
  - echo "$TEST"
```