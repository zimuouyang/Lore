```java
manifestPlaceholders
```

Future

```
CountDownLatch
```

多线程

- flavorDimensions

  在app:级别下的gradle文件中，加入productFlavors,并在productFlavors下创建产品A与B

  ```java
  productFlavors {
      //新建产品A
      A {
          //程序包名
          applicationId "com.wmj.a"
          //不同渠道号
          manifestPlaceholders = [UMC:"product-Complete"]
          //versionName
          versionName "1.0.0"
          //versionCode
          versionCode 1
  
      }
      //新建产品B
      B {
          //程序包名
          applicationId "com.wmj.b"
          //不同渠道号
          manifestPlaceholders = [UMC:"product-Temp"]
          //versionName
          versionName "2.1.1"
          //versionCode
          versionCode 2
      }
  
  ```

  使用flavorDimensions 组合不同环境

  ```java
  flavorDimensions "dev","app"
  productFlavors{
      //app2产品,注意(这边的产品名字字母不能大写)
     B{
        dimension "app"
        applicationIdSuffix "com.wmj.b"
        versionCode 1
        versionName "1.0.0"
      }
      //app1产品,注意(这边的产品名字字母不能大写)
     A{
        dimension "app"
        applicationId "com.wmj.a"
        versionCode 1
        versionName "1.0.0"
     }
      //线上环境
     product{
        dimension "dev"
     }
     //测试环境
     zeotest{
        dimension "dev"
  
     }
     //预发环境
     yufa{
        dimension "dev"
     }
  }
  
  ```

  