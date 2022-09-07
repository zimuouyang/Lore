- 将所有jar文件复制至某临时目录中，通过jar命令解压得到所有的.class文件
   jar -xvf xx.jar

  xx.jar必须为具体的jar，不能为*.jar，会报FileNotFoundException

- 删除临时目录下所有的jar文件
  del    /F   *.jar

- 合并所有.class文件至jar，需要切换至该临时目录，不然生成的jar会包含临时目录
   jar -cvfM game.jar .   //最后一点不要忘记了