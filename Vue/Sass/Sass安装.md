### 安装Ruby

​	在官网直接下载即可

```shell
ruby -v
//如安装成功会打印
ruby 2.6.4p104 (2019-08-28 revision 67798) [x64-mingw32]
```

### 安装RubyGems 

```shell
gem update --system //该命令请翻墙一下
```

### 安装Sass与Compass

```shell
//安装如下(如mac安装遇到权限问题需加 sudo gem install sass)
gem install sass
gem install compass

//安装成功可以使用以下命令验证
sass -v
Sass 3.x.x (Selective Steve)

compass -v
Compass 1.x.x (Polaris)
Copyright (c) 2008-2015 Chris Eppstein
Released under the MIT License.
Compass is charityware.
Please make a tax deductable donation for a worthy cause: http://umdf.org/compass
```

常用命令：

```shell
//更新sass
gem update sass

//查看sass版本
sass -v

//查看sass帮助
sass -h
```

https://www.sass.hk/install/  具体编译使用方式