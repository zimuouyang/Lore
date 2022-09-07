## 列出已安装和可用列表

```shell
sdkmanager --list
```

## 安装

```shell
sdkmanager [packages]


eg：
sdkmanager "platform-tools" "platforms;android-29" "build-tools;29.0.2"

```

下载的sdk将会安装到你的解压目录，也就是跟tools同一目录。
另外，platform-tools似乎不分版本，只会下载最新版，下载一次偶尔更新就行了。
下载过程中可能会询问是否同意协议，我们输入y然后回车即可继续安装：

## 更新

```
# 将会更新所有已安装的软件包
sdkmanager --update
```

## 卸载

卸载与安装差不多，添加一个`--uninstall`参数即可

```
sdkmanager --uninstall packages
```

