## ART和Dalvik
#### JVM与Dalvik
Android应用程序运行在Dalvik/ART虚拟机，并且每一个应用程序对应有一个单独的Dalvik虚拟机实例。Dalvik虚拟机实则也算是一个Java虚拟机，只不过它执行的不是class文件，而是dex文件。

Dalvik虚拟机与Java虚拟机共享有差不多的特性，差别在于两者执行的指令集是不一样的，前者的指令集是基本寄存器的，而后者的指令集是基于堆栈的。

|                   | Java Virtual Machine            | Dalvik Virtual Machine           |
| ----------------- | ------------------------------- | -------------------------------- |
| Instruction Set中 | Java Bytecode(Stack Based)      | Dalvik Bytecode(Register Based)  |
| File Fromat       | .class file(one file,one class) | .dex file(one file,many classed) |

那什么是基于栈的虚拟机，什么又是基于寄存器的虚拟机？

#### 基于栈的虚拟机
对于基于栈的虚拟机来说，每一个运行时的线程，都有一个独立的虚拟机栈。栈中记录了方法调用的历史，每有一次方法调用，栈中便会多一个栈桢。最顶部的栈桢称作当前栈桢，其代表着当前执行的方法。基于栈的虚拟机通过操作数栈进行所有操作。

比如这个方法：
```java
public static void test() {
        int a = 1;
        int b = 2;
        int c = a + b;
        throw new UnsupportedOperationException("hahahaha");
    }
```
短短的几行代码，在经过javac编译得到字节码后，我们来看它的执行是怎样的
```java
public static test()V
   L0
    LINENUMBER 5 L0
    ICONST_1
    ISTORE 0
   L1
    LINENUMBER 6 L1
    ICONST_2
    ISTORE 1
   L2
    LINENUMBER 7 L2
    ILOAD 0
    ILOAD 1
    IADD
    ISTORE 2
   L3
    LINENUMBER 8 L3
    NEW java/lang/UnsupportedOperationException
    DUP
    LDC "hahahaha"
    INVOKESPECIAL java/lang/UnsupportedOperationException.<init> (Ljava/lang/String;)V
    ATHROW
   L4
    LOCALVARIABLE a I L1 L4 0
    LOCALVARIABLE b I L2 L4 1
    LOCALVARIABLE c I L3 L4 2
    MAXSTACK = 3
    MAXLOCALS = 3
}
```
1. ICONST_1 :  将int类型常量1压入操作数栈；
2. ISTORE 0 :  将栈顶int类型值存入局部变量0；
3. ICONST_2 :  将int类型常量2压入操作数栈；
4. ISTORE 1 :  将栈顶int类型值存入局部变量1；
5. ILOAD 0  :  将局部变量0取出压入操作数栈
6. ILOAD 1  :  将局部变量1取出压入操作数栈
7. IADD : 执行int类型的加法 ；将操作数栈中两个变量相加，并压入操作数栈
8. ISTORE 2 ： 将栈顶int类型值存入局部变量2；

#### 基于寄存器的虚拟机

寄存器是CPU的组成部分。寄存器是有限存贮容量的高速存贮部件，它们可用来暂存指令、数据和位址。如下图所示，是一个寄存器操作的简易流程，由程序计数器，指令寄存器，ALU（算数逻辑单元），和数据寄存器（AX、BC和CX）组成。

指令寄存器首先取出第一条指令LOAD A,100，从100的内存地址中取出1放入AX中，然后再取出第二条指令LOAD B,104，从104的地址中取出1放入BX。接下来再取出ADD C,A,B指令，将AX中寄存的1和BX中寄存的1，放入ALU中执行加法操作，将ALU中运算的结果放入CX中。最后取出STORE C,108这条指令，将CX中的结果存入地址为108的内存空间中。

![](https://ae01.alicdn.com/kf/Ub4c2197bd17a4016b75a4e7818630b4aB.jpg)

基于寄存器的虚拟机中没有操作数栈，但是有很多虚拟寄存器。其实和操作数栈相同，这些寄存器也存放在运行时栈中，本质上就是一个数组。与JVM相似，在Dalvik VM中每个线程都有自己的PC和调用栈，方法调用的活动记录以帧为单位保存在调用栈上。在Dalvik虚拟机上中，有一块虚拟寄存器用来模拟真实的物理寄存器。
| Dalvik VM指令            |                                      |
| ------------------------ | ------------------------------------ |
| const/4 v0, #int 1 // #1 | 将1放到v0地址                        |
| const/4 v1, #int 2 // #2 | 将2放到v1地址                        |
| add-int v2, v0, v1       | 取出v0，v1地址的1和2相加，放入v2地址 |
| return-void              | 返回                                 |

同样是刚才的``test()``方法，与JVM版相比，可以发现Dalvik版程序的指令数明显减少了，数据移动次数也明显减少了。
#### ART与Dalvik
Dalvik虚拟机执行的是dex字节码，解释执行。从Android 2.2版本开始，支持JIT即时编译（Just In Time），在程序运行的过程中进行选择热点代码（经常执行的代码）进行编译或者优化。

而ART（Android Runtime） 是在 Android 4.4 中引入的一个开发者选项，也是 Android 5.0 及更高版本的默认 Android 运行时。ART虚拟机执行的是本地机器码。Android的运行时从Dalvik虚拟机替换成ART虚拟机，并不要求开发者将自己的应用直接编译成目标机器码，APK仍然是一个包含dex字节码的文件。那么，ART虚拟机执行的本地机器码是从哪里来？

**dex2aot**
Dalvik下应用在安装的过程，会执行一次优化，将dex字节码进行优化生成odex文件。而Art下将应用的dex字节码翻译成本地机器码的最恰当AOT时机也就发生在应用安装的时候。ART 引入了预先编译机制（Ahead Of Time），在安装时，ART 使用设备自带的 dex2oat 工具来编译应用，dex中的字节码将被编译成本地机器码。

**Android N的运作方式**
ART 使用预先 (AOT) 编译，并且从 Android N混合使用AOT编译，解释和JIT。

1、最初安装应用时不进行任何 AOT 编译（安装又快了），运行过程中解释执行，对经常执行的方法进行JIT，经过 JIT 编译的方法将会记录到Profile配置文件中。

2、当设备闲置和充电时，编译守护进程会运行，根据Profile文件对常用代码进行 AOT 编译。待下次运行时直接使用。

## Android类加载
#### 类加载机制
我们都知道编写的类是通过ClassLoader加载的，那么在Andoird中，Class类加载机制是怎样的呢。在java.lava包中，有一个ClassLoader的抽象类，这是所有类加载器的基类。

ClassLoader主要有两个子类，其中BootstrapClassLoader用于加载Android Framework层class文件。而BaseDexClassLoader中又有DexPathList、PathClassLoader和DexClassLoader等三个主要的子类。
- DexPathList
- PathClassLoader  　 Android应用程序类加载器
- DexClassLoader  　  额外提供的动态类加载器

![](https://ae01.alicdn.com/kf/Ufac4f359564c4570a80af97966227514m.jpg)
对于PathClassLoader来说，我们来学习它的类加载过程，在它的顶级父类ClassLoader中，有一个``loadClass()``方法来加载class，我们来看它的源码：
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                }  catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```
该方法需要传入两个参数，一个是class的全类名，第二个是。方法首先通过``findLoadedClass()``寻找类加载的缓存，如果没有则进入if。这里有一个parent对象，这个对象是父类加载器对象，然后就会调用父类加载器的``loadClass()``方法。如果父类加载器还有父类加载器的话，又会再调用父类加载器的父类加载器的``loadClass()``方法。只有父类加载器无法完成此加载任务或者没有父类加载器时，``才自己去findClass()``加载。这就是双亲分托机制。它有以下两个优点：

1. 避免重复加载，当父加载器已经加载了该类的时候，就没有必要子ClassLoader再加载一次。

2. 安全性考虑，防止核心API库被随意篡改。

对于PathClassLoader我们可以使用new PathClassLoader(ClassLoader parent)方法创建出来，可知需要传入ClassLoader对象，而在Android中，系统为PathClassLoader传入的是BootstrapClassLoader类加载器。

``findClass()``是在BaseDexClassLoader中，我们再看它的代码：
```java
public class BaseDexClassLoader extends ClassLoader {

    private final DexPathList pathList;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);

        if (reporter != null) {
            reportClassLoaderChain();
        }
    }

  @Override
  protected Class<?> findClass(String name) throws ClassNotFoundException {
      List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
      Class c = pathList.findClass(name, suppressedExceptions);
      if (c == null) {
          ClassNotFoundException cnfe = new ClassNotFoundException(
                  "Didn't find class \"" + name + "\" on path: " + pathList);
          for (Throwable t : suppressedExceptions) {
              cnfe.addSuppressed(t);
          }
          throw cnfe;
      }
      return c;
  }

}
```
在BaseDexClassLoader的构造方法中，传入了dex文件的路径，并初始化pathList对象。
```java
DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {

        ...

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);

        ...
    }
```
在``DexPathList()``方法中，因为一个dexPath下可能会有多个dex文件，首先通过``splitDexPath()``方法进行拆分，然后通过``makeDexElements()``生成Element数组，每一个dex就对应一个Element对象。
```java
public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```
然后在DexPathList的``findClass()``方法中，遍历刚才得到的Element数组，通过Element对象拿到对应的dexFile，然后使用``loadClassBinaryName()``寻找对应的Class对象。所以完整的类加载流程如下所示：

![](https://ae01.alicdn.com/kf/Ud01570cf1fed4ae89d95848497def8d8V.jpg)
#### 热修复
从前面的类加载机制我们知道，在DexPathList中会去遍历dexElements数组并寻找对应的Class，而热修复就是基于此。我们可以将我们的修复工程打包成.dex补丁包，然后插入到dexElements的前面去。

这样一来，当类加载器加载类的时候，就会先去加载我们补丁中的dex文件，并缓存Class。那么，我们要怎么去插入我们的.dex呢，无非就是反射机制嘛，基本流程如下。
- 获取到当前应用的PathClassLoader
- 反射获取到DexPathList属性对象pathList
- 反射修改pathList的dexElements
  1. 把补丁包patch.dex转化为Element[] （patch）
  2. 获得pathList的dexElements属性 （old）
  3. patch+old合并，并反射赋值给pathList的dexElements
