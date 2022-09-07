


没错，看了很多 ASM 入门的文章，都感觉文章写的很轻松，站立的高度都太高了，我个人觉得想要能够编写 ASM 相关代码，能看懂字节码是必不可少的，所以本文会以字节码为切入点，带大家简单的入门一下 ASM。

## Java Class 文件结构
大家都知道 *.java 文件经过 javac 编译之后会生成 *.class 文件，*.class 文件会被 Java 虚拟机进行加载。

Java 虚拟机之所有能够加载 class 文件，前提肯定是能够按照某种规则读取 class 文件的内容。

那么这个规则就是 *.class 文件的内部结构。

比如我们编写一个非常简单的 Java 类 Hello.java:
``` java
public class Hello{
	
	public static void main(String[] args){
		System.out.println(Hello.class.getSuperclass().getName());
	}
	
}
``` 

然后我们编译生成 Hello.class 文件。

这个时候我们来看下 Hello.class 文件结构，我们把 Hello.class 文件拖进一个支持 16 进制查看的编辑器（本文为 010 Editor）中。


可以看到整个文件都是二进制的代码，图中是以 16 进制的方式展示给大家，当然了这一对 16 进制的代码我们当然是看不懂的，不过如果你对 class 文件有一点熟悉，你可能知道开头这几个 16 进制字符CAFEBABE的含义。

这几个字符为 class 文件的魔数，大家都知道 Java 和咖啡有着一些秘密，通过图标也能看出来，所以魔术为 CAFEBABE 并不奇怪，好奇的可以去搜索下这个魔数的由来。

恩... 剩下的我们就很难看懂了...

当然你可以选择搜索「Java 文件格式」，这样你会找到非常详细的博文，来告诉你里面每段二进制代码的含义。

这样你就可以通过一篇详细的 class 文件格式的规范，来整理出其完整的面貌。

这是一件很枯燥的事情，不过我们是程序员。

如果 class 文件可以按照某种固定的格式来解析，那么我们不就可以写出一个程序来解析所有的 class 文件了吗？

没错，是这样的。

解析了之后，我们还可以将各个区域存放在我们设定的数据格式中，我们对外暴露接口去修改这些数据结构，最后按照 class 文件格式，再反向输出到文件。

这样，我们这套程序不但能够解析 class 文件，还能够修改 class 文件。

没错，当然了，这套程序我们都能想到，市面上肯定已经有成熟的方案了：

所以，我们今天文章的主角出来了：

ASM，就是其中一个非常成熟的开源库，它可以充当「这套程序」的角色，帮助我们解析 class 文件、修改 class 文件。

## 先来一个开胃菜，修改类的继承关系
你可能会疑惑，当我们熟悉了 class 文件夹的结构，做一下修改，就能够完成 class 文件的编辑吗？

没错。

下面我们演示一下 class 文件的修改，当然我不准备用 ASM，我们准备手工修改一下。

还是刚才的例子：
``` java
public class Hello{
	
	public static void main(String[] args){
		System.out.println(Hello.class.getSuperclass().getName());
	}
	
}
``` 
我们 main() 方法中输出了其父类的类名。

所以... 我准备修改掉 Hello 的父类，目前 Hello 继承自 Object。

按照我们前面的描述，我们只要能够找到这个 Hello.class 文件中表示其父类的字段区域，对它就行修改就行了。

恩，为了大家能够看明白我们怎么找到 class 文件的对应区域，我们可以借助 010 Editor，它内部有 class 文件模板，可以帮助我们较为清晰的看到每一部分的结构：


可以看到我们 class 文件在一系列的常量池之后，会包含访问修饰符，当前类名，以及父类名。

父类名对应的值为 7，7 代表了常量池中的第 7 个元素，我们找到第 7 个常量：


u1 tag =7代表这个是 Class 类型常量，常量值的索引为 25。

我们再往下看第 25 个常量：


长度为 16，字符数组为：106，97... 这一串数字。

这一串数字其实就是 ASCII 码，你可以随便找到个码表：


对应查出来，即为java/lang/Object。

历经这么多流程我们终于找到了对应父类的 16 进制代码的代码和编码了。

我们现在把 Hello 的继承类换成java/lang/Number。

那么只需要把 Object 对应的 16 进制代码换成 Number 就可以了。

换之前：


换之后：


详细的你可以看到 4F 换成了 4E，换算为 10 进制为：79，79 对应的 ASCII 码为 N，你可以按照如果规律，发现我们把 Object 换成了 Number。

然后我们保存后，执行一下：


在更换前后，我分别执行了 java，可以看到我们输出的父类名已经完成了更换。

我们甚至可以 javap 一下：


没错吧，确实继承关系发生了改变。

可以看到，只要我们能够找到指定区域，去修改这个区域的二进制代码，这样我们就能对 class 文件为所欲为。

是不是很简单。

不过事实上并不是那么简单。

因为我们修改的这个文件极其简单，如果内容非常多，发生字符串常量池复用的时候，我们就不能这么随意的修改某个常量池的内容了。

二来刚好java/lang/Number与java/lang/Object长度完全一致，否则我们还要做非常多的对齐工作。

所以，修改 class 文件也不是那么容易的事情。

别担心，我们有 ASM。

不过，即使有 ASM 这样的类库，也不代表我们不需要了解 class 文件就可以完成 class 文件的修改了。

最起码，从我来看 ASM 类库的本意并不像 hibernate 那样，让不了解 sql 的开发者也能写出操作数据库的代码。

我们还是要了解 class 文件内部的组成，并且在修改代码时，我们还要了解代码执行时，局部变量表是如何工作的，指令对栈帧的影响等等。

我们往下看，你就明白了。

## 开始引入 ASM
### 引入 ASM
好了，下面我们开始正式学习 ASM。

首先我们找到 ASM 的官网：

asm.ow2.io/

在官网你可以看到目前最新的版本，还有一份详细的 User guide，基本包含了所有 API 的介绍。

看官网上版本迭代目前已经跟新到 9.1 了，那就试用最新版本吧：

// https://mvnrepository.com/artifact/org.ow2.asm/asm-commons
implementation group: 'org.ow2.asm', name: 'asm-commons', version: '9.1'
复制代码
尝试分析 Class 文件
从学习的角度来说，在修改 class 文件之前，我们可以先学习下怎么读取 class 文件内部的各个部分。

比如我想在编译期间通过编译的 *.class 的文件，获取其内部的所有方法名称，字段名称。

Tree Api
对于分析 class 文件，我们最希望的方式是什么？

肯定是：我给你个 class 文件，然后你给我返回个 ClassNode 对象，这个对象最好有个 类似 List<Methood>，List<Field > 这样的方法或者字段。

恩... 想得倒美，ASM 是这么简单的东西吗？

不过，ASM 还真就这么简单，我们来看一个类 ClassNode，这个类上注释是：

A node that represents a class.
复制代码
用来指代一个 class 文件。

那么我们想要的 class 内部的一切，应该可以通过这个类的 API 直接或者间接的获取。

没错，是的，看一眼：


我们只要通过 class 文件构造这么个 ClassNode 对象，好像就可以为所欲为了。

来看下代码把：

首先我们编写一个 User:
``` java
public class User {
    private String name;
    private int age;
    public String getName() {
        return name;
    }
    public int getAge() {
        return age;
    }
}
``` 
然后我们希望获取 User.class 中包含的所有方法以及字段：
``` java
public class TreeApiTest {

    public static void main(String[] args) throws Exception {
        Class clazz = User.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));
        ClassNode classNode = new ClassNode(Opcodes.ASM5);
        classReader.accept(classNode, 0);

        List<MethodNode> methods = classNode.methods;
        List<FieldNode> fields = classNode.fields;

        System.out.println("methods:");
        for (MethodNode methodNode : methods) {
            System.out.println(methodNode.name + ", " + methodNode.desc);
        }

        System.out.println("fields:");
        for (FieldNode fieldNode : fields) {
            System.out.println(fieldNode.name + ", " + fieldNode.desc);
        }

    }

}
``` 

上述代码有个辅助类 Utils.getClassFilePath 方法我贴一下，主要是可以通过 Class 对象，找到其在 AS 中具体的路径。
``` java
public static String getClassFilePath(Class clazz) {
        // file:/Users/zhy/hongyang/repo/BlogDemo/app/build/intermediates/javac/debug/classes/
        String buildDir = clazz.getProtectionDomain().getCodeSource().getLocation().getFile();
        String fileName = clazz.getSimpleName() + ".class";
        File file = new File(buildDir + clazz.getPackage().getName().replaceAll("[.]", "/") + "/", fileName);
        return file.getAbsolutePath();
    }
``` 
看下我们代码的流程：

首先我们拿到 class 文件的路径；
然后交给 ClassReader
再构造一个 ClassNode 对象
调用 ClassReader.accept() 方法完成对 class 遍历，并把相关信息记录到 ClassNode 对象中；
这个时候，我们就能够通过 ClassNode 去拿我们所想要的信息了，看一下输出：
``` groovy
methods:
	<init>, ()V
	getName, ()Ljava/lang/String;
	getAge, ()I
fields:
	name, Ljava/lang/String;
	age, I

``` 
到这里，有没有发现，如果只是读取 class 文件，是不是简单得不能再简单了？

上述的 API，称为 Tree Api，即我们分析完成 class 文件，把信息存储到 ClassNode，然后通过 ClassNode 再读取即可，有点类似 xml 文件解析时，把整个 xml 文件读取到内存中的方式。

Core Api
不过大家如果看博客，其实上述写法在博客中并不多见，更多的博客上书写的还是基于 “事件驱动” 的 API，即解析 class 文件过程中，每遇到一个 “节点”，把节点信息交给你，我们类似于监听“节点” 的解析事件，我们看下代码：
``` java
public class VisitApiTest {
    public static void main(String[] args) throws Exception {
        Class clazz = User.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));
        ClassVisitor classVisitor = new ClassVisitor(Opcodes.ASM5) {
            @Override
            public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
                System.out.println("visit field:" + name + " , desc = " + descriptor);
                return super.visitField(access, name, descriptor, signature, value);
            }

            @Override
            public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
                System.out.println("visit method:" + name + " , desc = " + descriptor);
                return super.visitMethod(access, name, descriptor, signature, exceptions);
            }
        };
        classReader.accept(classVisitor, 0);
    }
}
``` 
我们看下输出：
``` grovvy
visit field:name , desc = Ljava/lang/String;
visit field:age , desc = I
visit method:<init> , desc = ()V
visit method:getName , desc = ()Ljava/lang/String;
visit method:getAge , desc = ()I
``` 
我们再次梳理下步骤：

首先我们拿到 class 文件的路径；
然后交给 ClassReader；
再构造一个 ClassVisitor 对象；
将 ClassVisitor 对象传入 ClassReader.accept()方法来接受对 class 文件解析时的 “节点” 回调信息；
可以看到，其实和上面的 Tree Api 还是比较类似的，一个是解析完成了，将所有信息保存到一个具体的对象，我们去读取；一个是参与到解析的流程中，监听解析节点的回调，输出结构。

稍微脑洞一下，假设我们自定义一个 ClassVisitor，在 visitMethod 方法中，把监听到的方法声明信息存储下来，放到一个 List 结合，那么我们是不是我们就实现了一个简单的ClassNode实现呢？

所以 ClassNode 也是 ClassVisitor 的子类？

没错！
``` java
public class ClassNode extends ClassVisitor 
``` 
到这里，你应该已经掌握了如何去读取一个 class 文件的内容：

class-> ClassReader.accept(ClassVisitor)

甚至一不小心掌握了 ClassNode 的原理，真是太开心了，那，我们继续。

简单修改下字节码
修改字节码，就不会这么简单了哈，做好心理准备。

我们看个官方的例子：
``` java
public class C {
      public void m() throws Exception {
         Thread.sleep(100);
      }
}
``` 
修改为：
``` java
public class C {
    public static long timer;

    public void m() throws Exception {
        timer -= System.currentTimeMillis();
        Thread.sleep(100);
        timer += System.currentTimeMillis();
    }
}
``` 

有点像我们给添加耗时操作的代码。

我们前面学习了，如何读取、遍历解析一个 class 文件，我们还没有尝试过如果回写一个 class 文件，即我们对 class 文件做了一些修改，我们要尝试写回去覆盖原来的 class 文件。

初识 ClassWriter
看代码前，我们想一下，如果让我们来实现读取后写入，你会怎么做：

我们首先遍历的时候，拿到了 class 中所有的信息，然后调用个 writeToFile 方法，直接写入不就行了么。

所以 ClassWriter 就是个 ClassVisitor？

他做的事情，就是遍历的时候保存信息，然后支持按照 class 文件格式写文件就行了。

恩... 猜测正确的七七八八了。

我们先看下 ClassWriter 是不是 ClassVisitor:

public class ClassWriter extends ClassVisitor

我脑子里面已经有接下来代码的画面了：
``` java
public class ClassWriterTest {

    public static void main(String[] args) throws Exception {
        Class clazz = C.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));

        ClassWriter classWriter = new ClassWriter(0);
        classReader.accept(classWriter, 0);

        // 写入文件
        byte[] bytes = classWriter.toByteArray();
        FileOutputStream fos = new FileOutputStream("/Users/zhy/Desktop/copyed.class");
        fos.write(bytes);
        fos.flush();
        fos.close();

    }
}
``` 
上述代码，我们就完成了一个类的复制，如果我们传入相同的路径，那就完成了类的修改（当然了，目前我们还没修改）。

原理刚才都猜过了，唯一不同的就是没有 writeToFile，而是有个 toByteArray 方法，毕竟 byte 数组更加通用一些。

开始尝试添加字段
虽然我们学会了 ClassWriter 的基础用法，但是我好像发现了一个问题。

ClassReader.accept 方法，只能接受一个 ClassVisitor 对象，因为我们必须要修改字节码，所以 ClassWriter 肯定是要传入的。

那么我们怎么传入另一个 ClassVisitor 对象去修改字节码呢？

我们看一个 ClassVisitor 的构造方法：
``` java
public ClassVisitor(final int api, final ClassVisitor classVisitor) 
``` 
是不是瞬间明白了，我们可以给 ClassVisitor 传入一个实际对象，自己作为代理对象，需要拦截的方法，我们复写做操作，有点类似 ContextWrapper。

首先我们尝试添加一个字段，如果是 Tree API，其实通过 ClassNode 去 add 一个 FieldNode 就可以了。

不过我们选择用 visit API 来实现，我们先编写一个 ClassVisitor 的子类：
``` java
public class AddTimerClassVisitor extends ClassVisitor {

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }   
}
``` 
这个时候，我们先代理下真正的 ClassVisitor 对象，我们要做的就是：

找个合适的位置插入一个字段；

剩下的都交给传入的 ClassVisitor 去做。

那么什么是合适的位置呢？

其实就是复写哪个方法了，那我们得知道 ClassVisitor 大概会执行哪些方法，这些方法的执行顺序是什么样子的：
``` grovvy
visit visitSource? visitOuterClass? ( visitAnnotation |
   visitAttribute )*
   ( visitInnerClass | visitField | visitMethod )*
   visitEnd
``` 
可以看到 ClassVistor 在遍历一个类的时候，相关调用顺序如上，? 代表这个方法可能不会调用，* 标识可能会调用 0 次或者多次。

那么我们这个合适的位置，首先要选择一定会调用的地方；其次最好能只执行一次；最后因为本身其内部可能就有 field，我们可以收集信息，防止插入重名 field 的情况，所以最终我们选择 visitEnd 方法中。

选择了方法，那么如何才能插入一个 field 呢？

我们思考下：

其实我们的 class 最终是由 ClassWriter 去生成的，它会通过 visitField 去收集相关信息，也就是说，你调用一次 ClassWriter.visitField 方法，他就会以为真有这个 field，然后记录下来。

那就简单了，我们看代码：
``` java
public class AddTimerClassVisitor extends ClassVisitor {

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }


    @Override
    public void visitEnd() {

        FieldVisitor fv = cv.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "timer",
                "J", null, null);
        if (fv != null) {
            fv.visitEnd();
        }
        cv.visitEnd();
    }
}
```
demo 代码，实际生产环境需要做更严格的条件校验。

我们给原类添加了一个 timer 字段，访问修饰符是 public static，并且其类型是 J 也就是 long 类型。

我们把代码组装到一起：
``` java
public class ClassWriterTest {

    public static void main(String[] args) throws Exception {
        Class clazz = C.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));

        ClassWriter classWriter = new ClassWriter(0);
        
        AddTimerClassVisitor addTimerClassVisitor = new AddTimerClassVisitor(Opcodes.ASM5, classWriter);
        classReader.accept(addTimerClassVisitor, 0);

        // 写入文件
        byte[] bytes = classWriter.toByteArray();
        FileOutputStream fos = new FileOutputStream("/Users/zhy/Desktop/copyed.class");
        fos.write(bytes);
        fos.flush();
        fos.close();

    }
}
``` 
复制代码
然后我们运行下。

运行生成的类，反编译看一下：


开心...

接下来我们要尝试修改字节码了，为方法添加耗时信息打印了。

修改方法
通过上文的学习，我们之前对于方法的遍历，会执行 ClassVisitor 的 visitMethod 方法，修改方法肯定是离不开这个方法了，所以我们详细的看下这个方法：
``` java
# ClassVisitor
public MethodVisitor visitMethod(
      final int access,
      final String name,
      final String descriptor,
      final String signature,
      final String[] exceptions) {
    if (cv != null) {
      return cv.visitMethod(access, name, descriptor, signature, exceptions);
    }
    return null;
  }
``` 
可以看到，这个方法的参数中包含了方法所有声明相关的信息，但是没有包含实际运行的代码相关信息，即指令信息。

不过可以看到，这个方法的返回值并不是 null，而是一个 MethodVisitor，所以我们 ClassReader 遍历 class 文件的思路肯定是：先给你方法声明相关信息，然后我们给它返回一个 MethodVisitor，它拿到这个 MethodVisitor，再通过 MethodVisitor 开始遍历这个方法内部的所有信息。

所以... 我们需要自定义一个 MethodVisitor 完成代码的插入。

先撸一点代码：
``` java
public class AddTimerClassVisitor extends ClassVisitor {

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {

        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);

        MethodVisitor newMethodVisitor = new MethodVisitor(api, methodVisitor) {
           
        };
        
        return newMethodVisitor;
    }
``` 
我们在刚才的 AddTimerClassVisitor 中复写了 visitMethod，再其内部我们自定义了一个 MethodVisitor 代理一波本来的对象。

问题又来了？通过举一反三的思想，我们应该能够猜到 MethodVisitor 跟 ClassVisitor 设计应该是类似的，里面一堆 visitXXX 方法，我们这次修改字节码是在方法前后分别注入代码，那么到底该选择复写哪些方法呢？

这就要求我们知道 MethodVisitor 中各种 visitXXX 方法的执行顺序了：
``` grovvy
visitAnnotationDefault?
(visitAnnotation |visitParameterAnnotation |visitAttribute )* ( visitCode
(visitTryCatchBlock |visitLabel |visitFrame |visitXxxInsn | visitLocalVariable |visitLineNumber )*
visitMaxs )? visitEnd
``` 
首先是遍历一些注解、参数相关信息；从 visitCode 开始遍历一整个方法。

我们的注入是：

方法开始：我们选择复写 visitCode 方法；
RETURN 之前：我们选择复写 visitXxxInsn，再其内部判断当前指令是否是 RETURN；
选择好了注入的时机，问题来了，我们好像还不知道注入的代码怎么写呢？

是的，这里其实要求大家对字节码是有足够的掌握，不然我怎么写估计都不太好理解，但是我尽量用推导的方式，引导大家去理解。

首先我们要了解我们添加的代码，会以字节码指令的方式注入进去，所以我们要先大概看下在字节码的层面上变化是怎样的。

所以修改之前，我们要看分别看一下修改前与修改后对应的方法字节码：
``` grovvy
public void m() throws Exception {
       Thread.sleep(100);
      }
``` 
对应字节码：
``` 
public void m() throws java.lang.Exception;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: ldc2_w        #2                  // long 100l
         3: invokestatic  #4                  // Method java/lang/Thread.sleep:(J)V
         6: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/imooc/blogdemo/blog03/C;
    Exceptions:
      throws java.lang.Exception
}
``` 
``` java
public void m() throws Exception {
        timer -= System.currentTimeMillis();
        Thread.sleep(100);
        timer += System.currentTimeMillis();
    }
```
对应字节码：


我框起来这两块代码，就是我们新增的字节码指令，那么我们实际要编写的代码和这两块新增的字节码是一一对应的。

先看我们方法最前面添加的指令：
``` java
@Override
public void visitCode() {
    mv.visitCode();

    mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
            "currentTimeMillis", "()J");
    mv.visitInsn(LSUB);
    mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");

}
``` 
你仔细观察一下，其实和我们框起来的字节码对比：
``` grovvy
0: getstatic     #2                  // Field timer:J
3: invokestatic  #3                  // Method java/lang/System.currentTimeMillis:()J
6: lsub
7: putstatic     #2                  // Field timer:J
``` 
基本上没有任何区别。

不过我们还是解释下这几行字节码：
``` 
拿当前类静态变量 timer，压到操作数栈
调用 System. System.currentTimeMillis，方法返回值压到操作数栈；
调用 "timer - System. System.currentTimeMillis"，结果压栈
将 3 得到的值，再次赋值给 timer 字段；
``` 
换成代码其实就是：
``` java
timer -= System.currentTimeMillis();
``` 
复制代码
同样的，我们把方法 RETURN 前的代码也写了：
``` java
@Override
public void visitInsn(int opcode) {

    if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
        mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                "currentTimeMillis", "()J");
        mv.visitInsn(LADD);
        mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");
    }
    mv.visitInsn(opcode);
}
``` 
你可以采用和上面相同的方式去对比字节码。

有一点不同的是，对于 RETURN 这个指令，我们判断了多个，因为我们并不知道当前方法的返回值情况，如果确定方法没有返回值，那么只要判断 RETURN 即可。

好了，我们贴下完整的代码：
``` java
package com.imooc.blogdemo.blog03;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

import static org.objectweb.asm.Opcodes.*;

public class AddTimerClassVisitor extends ClassVisitor {


    private String mOwner;

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
        mOwner = name;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {

        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);

        if (methodVisitor != null && !name.equals("<init>")) {
            MethodVisitor newMethodVisitor = new MethodVisitor(api, methodVisitor) {
                @Override
                public void visitCode() {
                    mv.visitCode();

                    mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                            "currentTimeMillis", "()J");
                    mv.visitInsn(LSUB);
                    mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");

                }

                @Override
                public void visitInsn(int opcode) {

                    if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
                        mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
                        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                                "currentTimeMillis", "()J");
                        mv.visitInsn(LADD);
                        mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");
                    }
                    mv.visitInsn(opcode);

                }
            };
            return newMethodVisitor;
        }

        return methodVisitor;
    }

    @Override
    public void visitEnd() {

        FieldVisitor fv = cv.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "timer",
                "J", null, null);
        if (fv != null) {
            fv.visitEnd();
        }
        cv.visitEnd();
    }
}

```
注：这其实就是官方文档中的一个例子。

然后我们运行一下，在桌面生成 copyed.class 文件。

完美，反编译一下，正常。


这个时候，还不能开心的太早，可以反编译不代表你代码编写就没有错误！

我们来验证下文件的正确性：

简单修改了一下输出文件的路径：
``` 
byte[] bytes = classWriter.toByteArray();
// 修改，为了一会能 java 执行
FileOutputStream fos = new FileOutputStream("/Users/zhy/Desktop/com/imooc/blogdemo/blog03/C.class");
fos.write(bytes);
fos.flush();
fos.close();
``` 
然后我们切到桌面，执行下：
``` 
java com.imooc.blogdemo.blog03.C
``` 
``` c++
192:Desktop zhy$ java com.imooc.blogdemo.blog03.C
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main" java.lang.VerifyError: Operand stack overflow
Exception Details:
  Location:
    com/imooc/blogdemo/blog03/C.m()V @3: invokestatic
  Reason:
    Exceeded max stack size.
  Current Frame:
    bci: @3
    flags: { }
    locals: { 'com/imooc/blogdemo/blog03/C' }
    stack: { long, long_2nd }
  Bytecode:
    0x0000000: b200 12b8 0018 65b3 0012 1400 19b8 0020
    0x0000010: b200 2412 26b6 002c b200 12b8 0018 61b3
    0x0000020: 0012 b1                                

	at java.lang.Class.getDeclaredMethods0(Native Method)
	at java.lang.Class.privateGetDeclaredMethods(Class.java:2701)
	at java.lang.Class.privateGetMethodRecursive(Class.java:3048)
	at java.lang.Class.getMethod0(Class.java:3018)
	at java.lang.Class.getMethod(Class.java:1784)
	at sun.launcher.LauncherHelper.validateMainClass(LauncherHelper.java:544)
	at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:526)
``` 
报错了！我们讲的头头世道，结果一运行，啪啪打脸。

为什么呢？

我们观察下报错原因：Exceeded max stack size.

看起来我们忘记了一个细节。

什么细节呢？

我们需要回归下，刚才代码修改前后的字节码，其中有个细节我们忽略了：
``` java
// 前
public void m() throws java.lang.Exception;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1

// 后
public void m() throws java.lang.Exception;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=4, locals=1, args_size=1
``` 
大家注意到没有，方法有个 stack 值变化了！

这个值是什么意思呢？

他指的是我们操作数栈需要的深度，我们的 Java 字节码指令，其中很多指令操作都会压栈操作，然后有些指令或者方法调用会弹出栈中的操作数去执行，例如 lsub 就会弹出操作数栈里面的两个 long 类型的值去做减法，完成后再把结果压栈。

在这个方法中所有的指令执行完成，所需要的栈的深度在编译期就确定了，从反编译结果来看我们需要变成 4。
``` java
@Override
public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);
        MethodVisitor newMethodVisitor = new MethodVisitor(api, methodVisitor) {
            @Override
            public void visitCode() {
                // 省略
            }

            @Override
            public void visitInsn(int opcode) {
               // 省略
            }

            @Override
            public void visitMaxs(int maxStack, int maxLocals) {
                mv.visitMaxs(4, maxLocals);
            }
        };
        return newMethodVisitor;
}
``` 
再次运行，生成 C.class，然后再次 java 一下:
``` 
Desktop zhy$ java com.imooc.blogdemo.blog03.C
错误: 在类 com.imooc.blogdemo.blog03.C 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
``` 
这次的错误终于正常了，因为我们没有写 main() 方法。

不过，事情并没有结束。

我们把 maxStack 写成 4 合理吗？

mv.visitMaxs(4, maxLocals);
复制代码
明显不合理，我们只是针对我们这个单一的方法，有可能之前某个方法 maxStack 是 10。

所以这个修改是极度不合理的，那么我们是不清楚经过我们的修改到底给多少合适的，所以我们可以定义一个增量值。

在我们刚才修改的字节码中：
``` grovvy
getstatic timer // 压栈 2
invoke System.currentTimeMillis // 压栈 2
LSUB // 出栈4，压栈 2
put static timer // 出栈 2
``` 
你可以按照上述也分析下 RETURN 之前插入的代码，最大的压栈数字是 4，也就说我们最大会增加 4 个操作数栈的深度。

你可能会有疑问，为什么 getstatic timer，压栈是 2 不是 1 吗？

因为 long 类型占两个位置。

所以我们应该写成：
``` java
@Override
public void visitMaxs(int maxStack, int maxLocals) {
    mv.visitMaxs( maxStack + 4, maxLocals);
}
``` 

然后验证一下，也没问题。

不过 stack 变成了 6，实际上 4 就足够了。


所以有没有更好的方式呢？

有的！

我们在构建 ClassWriter 的时候，代码是这么写的：
``` java

ClassWriter classWriter = new ClassWriter(0);
``` 

注意构造方法传入了一个 0，实际上接受的是一个 flag，其实有个 flag 是：

ClassWriter.COMPUTE_FRAMES

我们可以构建时传入：
``` java
ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
``` 

这样就可以删除刚才的 visitMaxs 相关代码了，它会自动帮我们重新计算 stackSize。

这个时候，你再重新运行，再次反编译，又可以看到 stack =4 啦。

到这里，初步的一个字节码修改案例就到这了。

可以看出来，如果你想通过 ASM 修改 class 文件，最起码你得：

字节码指令要非常清楚；
了解操作数栈；
当然，不仅如此，你还应该了解局部变量表，以及一些在编译过程中背后的原理。

只不过目前的例子还没接触到，其他细节，我们有缘再写。
