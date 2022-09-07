# Class 文件 
## Class 文件是啥
编译后被 Java 虚拟机所执行的代码使用了一种平台中立（不依赖于特定硬件及操作系统的）的二进制格式来表示，并且经常（但并非绝对）以文件的形式存储，因此这种格式被称为 Class 文件格式。Class 文件格式中精确地定义了类与接口的表示形式。 -- 来自 Java 虚拟机规范

我们知道，高级语言程序被执行需要先进行翻译程序，再进行执行。翻译程序分编译程序与解释程序，编译与解释最大的不同是解释程序没有目标代码的生成。而 Java 是一门编译性语言（也有说是解释性的），在执行的过程中会生成中间代码 class 文件。class 文件能够被 jvm 虚拟机所载入和执行，这样子就能够实现 Java“与平台无关” 的特性。

Java 虚拟机不和包括 Java 在内的任何语言绑定，它只与 “Class 文件” 这种特定的二进制文件格式所关联，比如 Python，Scala 文件都能生成 Class 文件。Class 文件中包含了 Java 虚拟机指令集和符号表以及若干其他辅助信息。
乱七八糟的说那么多看不懂的接下来进入正题。

## 文件结构
首先 class 文件与 dex 文件都是 8 字节二进制文件流，也就是都是由 0 和 1 组成，我们可以用 010Editor（强烈安利）这个工具打开。那问题来了，大于 8 字节的数据项怎么存呢？当需要占用 8 位字节以上空间的数据项时，则会按照高位在前的方式分割成若干个 8 位字节进行存储。（注意区分 8 字节与 8 位的区别，1byte=1bit）

在 Java 虚拟机规范的第四章中有规定 class 的文件格式：

使用 u1，u2 和 u4(分别代表了 1、2 和 4 个字节的无符号数) 来表示 Class 文件的内容

采用类似 c 语言结构体的伪结构来描述 Class 文件格式，把描述类结构格式的内容定义为项，这种伪结构中只有两种数据类型：无符号数和表。在 Class 文件中，各项按照严格顺序连续存放的，它们之间没有任何填充或对齐作为各项间的分割符号。

表（ Table） 是由任意数量的可变长度的项组成，用于表示 Class 文件内容的一系列复合结构。在 Class 文件中，表习惯性的使用_info 结尾，下文会举详细例子。

Class 文件呢 其实是一个大表~

## Class 文件的生成
要讲 Class 文件的结构，肯定得打开 Class 文件研究下，所以讲下 class 文件的生成。在最开始学 Java 的时候一般都会使用命令行来编译 Java 文件，使用的是 javac 和 java 命令，而使用 javac 命令就能生成 class 文件。简单的写个 HelloWorld.java

public class HelloWorld{
    public static void main(String[] args){
    System.out.println("hello world");
    }
}
$:javac HelloWorld.java target 1.6 source 1.6
-target <release>            生成特定 VM 版本的类文件  
-source <release>            提供与指定发行版的源兼容性

---
## 深入 Class 文件的内部数据结构
用 010Editor 打开刚刚生成的 HelloWorld.class 文件：

为了方便查看，010Editor 将二进制转换成了十六进制表示，毕竟看一堆 01 可能会发蒙。上面的十六进制是该 Class 文件的内容。

看下面那些内容（上面的也看不懂啊 (#`O′)。可以看到很多个 struct 这就是前面提到的伪结构了，最上面的 struct ClassFile 就是那个 “大表 "。

每一个 Class 文件对应于一个如下所示的 ClassFile 结构体。
``` java
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];
    u2 fields_count;
    field_info fields[fields_count];
    u2 methods_count;
    method_info methods[methods_count];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
``` 
来一个个的解释这些项的含义。

##  magic
魔数，是 u4 类型， 魔数的唯一作用是确定这个文件是否为一个能被虚拟机所接受的 Class 文件。 魔数值固定为 0xCAFEBABE（咖啡宝贝~）， 不会改变。看上面那张图的开头 4 字节也验证了这一点。其实不止 Class 文件中有这项数据，其他文件比如图片文件也是有的，就是可能叫法不同。

## minor_version、major_version
副版本号和主版本号， minor_version 和 major_version 的值分别表示 Class 文件的副、 主版本。 它们共同构成了 Class 文件的格式版本号。 譬如某个 Class 文件的主版本号为 M，副版本号为 m，那么这个 Class 文件的格式版本号就确定为 M.m。在我们的 HelloWorld.class 中，次版本号是 0x00, 主版本号是 0x32，即 50.0（16 进制的 32 转换成 10 进制是 50，在 010Editor 中 Vlaue 那一项可以查看）。一个 Java 虚拟机实例只能支持特定范围内的主版本号（ Mi 至 Mj） 和 0 至特定范围内 （ 0 至 m）的副版本号。不同版本的 Java 虚拟机实现支持的版本号也不同，高版本号的 Java 虚拟机实现可以支持低版本号的 Class 文件。

## constant_pool_count
常量池常量计数器，一开始看到我还以为是常量池的数量来着~ 其实是常量的数量。为什么需要有这一项？

表是由可变长数据组成的复合结构（ 表中每项的长度不固定），因此无法直接将字节偏移量来作为索引对表进行访问（如果采用这种方式，假设我们需要访问第 5 个常量，那就需要计算前四个常量的偏移量之和才能访问第 5 个常量）。 而我们描述一个数据结构为数组时，就意味着它含有零至多个长度固定的项组成，这个时候则可以采用数组索引的方式来访问它。数组需要知道它的长度吧，这个 constant_pool_count 可以看成是这个数组的长度。我们看前面的图也能发现每个 struct cp_info 后面都标明了 constant_pool[index]，在后面我们也有很多类似的结构，都是同样的道理。

当需要描述同一类型但数量不多的数据时，经常会使用一个前置的容量计数器加若干个连续的数据项的形式。

constant_pool_count 的值等于 constant_pool 表中的成员数加 1。在 HelloWorld.class 中，constant_pool_count 是 29，但是常量只有 28 项。这又是为啥！

正常的计数习惯来讲：大小是 29，下标应该是 028。但是在这里计数是从第 1 项开始的，也就是 128。把第 0 项空出来了，这样子可以满足某些指向常量池的索引值的数据在特定情况下需要表达 “不引用任何一个常量池项目” 的含义。

## constant_pool[]
常量池， constant_pool 是一种表结构 ，它包含 Class 文件结构及其子结构中引用的所有字符串常量、 类或接口名、字段名和其它常量。常量池中每一项又是一个表，这些表的结构不相同。但是有一个共同点就是第一项都是 tag 标志位

下面这个表应该是现在最新的常量池项目类型了，献上官网。value 是 tag 的值

常量池项目类型. png
不到黄河心不死，来验证下表中内容。下图可以看到第一个常量的 tag 值是 10，是 CONSTANT_Methodref 类型，正好跟上面的表对应上了~ 而 class_index 是指向常量池中一个 CONSTANT_Class 类型的常量，表示声明该方法的类描述符的索引项。这里值是 6，即第 6 项。由于 010Editor 对常量池是从 0 开始编号，所以我们应该找下标为 5 的常量，可以看到是 CONSTANT_Class 类型。

而 CONSTANT_Class 类型中的 name_index 是指向一个 CONSTANT_Utf8 类型的常量，表示该类的全限定名。
可以如下使用命令查看 class 文件的内容.

$ javap -verbose HelloWorld
下图就是运行该命令后的内容，这里主版本是 53 的原因是我又编译了一次这个 Java 类但是忘了指定 target 参数造成的。


## access_flags
访问标志， access_flags 是一种掩码标志， 用于表示某个类或者接口的访问权限及基础属性。取值范围和意义如下：


在上表中没有使用的 access_flags 标志位是为未来扩充而预留的，这些预留的标志为在编译器中会被设置为 0， Java 虚拟机实现也会自动忽略它们。
比如 ACC_MODULE 是 Java SE9 加进来的~

## this_class、super_class
类索引， this_class 的值必须是对 constant_pool 表中项目的一个有效索引值。constant_pool 表在这个索引处的项必须为 CONSTANT_Class_info 类型常量，表示这个 Class 文件所定义的类或接口。父类索引，对于类来说， super_class 的值必须为 0 或者是对 constant_pool 表中项目的一个有效索引值。 比如图 6 中红色圈出来的：this_class 指向 #5，#5 的值是 HelloWorld；super_class 指向 #6，#6 的值是 java/lang/Object。

## interfaces_count
接口计数器， interfaces_count 的值表示当前类或接口的直接父接口数量（对于类是实现，对于接口是继承）。

## interfaces[]
接口表， interfaces[] 数组中的每个成员的值必须是一个对 constant_pool 表中项目的一个有效索引值， 它的长度为 interfaces_count。如果没有实现接口，这一项就没有~

## fields_count
字段计数器， fields_count 的值表示当前 Class 文件 fields[] 数组的成员个数。fields[] 中每一项都是一个 field_info 结构的数据项， 它用于表示该类或接口声明的类字段或者实例字段（成员变量和实例变量）。

## fields[]
字段表， fields[] 数组描述当前类或接口声明的所有字段，但不包括从父类或父接口继承的部分。

## methods_count
u2 类型，方法计数器， methods_count 的值表示当前 Class 文件 methods[] 数组的成员个数，即方法的个数，所以一个类中如果方法个数超过了 65535，会编译失败。（难道这就是传说中 Android 的 65535 问题？可能有点原因...

## methods[]
方法表，methods[] 中每一项都是一个 method_info 结构的数据项，用于表示当前类或接口中某个方法的完整描述（包括方法名，方法签名，方法修饰符，方法中的代码）。methods[] 数组只描述当前类或接口中声明的方法，不包括从父类或父接口继承的方法。

## access_flags 描述的是方法的修饰符；name_index 描述的是方法名的简单名称；

## description_index 是方法的描述符。
描述符的作用是用来描述字段的数据类型、方法的参数列表和返回值。比如 method[1] : public static void main(String[] args) 描述符是 ([Ljava/lang/String;) V 。（）表示是个方法，参数是 String[] , 返回值类型是 void。

## attributes_count：
attribute_info 表的属性数量，具体下面会再提到~
方法表可以对照着下面两张图看（注意只有常量池是从 1 开始计数，包括前面的字段和接口表也都是 0 开始计数）：

## attributes_count、attributes[]
属性计数器、属性表。这一段好长。。我感觉这一块是最复杂的结构了属性表里面有属性表里面还有属性表。



