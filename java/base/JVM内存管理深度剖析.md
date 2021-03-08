
## JVM
#### 什么是JVM
JVM全称为Java Virtual Machine，就是Java虚拟机。我们的Java程序，经过javac编译之后，会生成Java字节码，通常是.class、.jar等，JVM把字节码翻译之后，就会生成操作系统能识别的机器语言。JVM有一个很重要的特性，那就是跨平台，在我们下载JDK的时候，就会有相应平台的JDK版本，不管是Windows，Linux还是Mac，都有其适配的JVM，使用不同平台的jdk，JVM就能生成在该平台可以运行的机器代码，同一个HelloWorld程序，在上面运行的效果都是一样的。

JVM还有一个重要的特性，那就是跨语言，像Kotlin、Groovy、Scala这些语言，都可以编译成为遵守规范的字节码，这些字节码都可以在Java虚拟机上运行。Java虚拟机不关心这个字节码是不是来自于Java程序，只需要各个语言提供自己的编译器，字节码遵循字节码规范，比如字节码的开头是CAFEBABY。

在JVM中，有一个Java类加载器（ClassLoader），生成的字节码（.class文件）会被加载到一个称为运行时数据区的地方，这个就是JVM管理的内存。而类中还有一些方法，就会交由执行引擎，通过解释执行或者JIT，调用操作系统去执行。

#### 运行时数据区
**定义**
Java虚拟机在执行Java程序的过程中，会把它所管理的内存划分为若干个不同的数据区域。而这些区域又可细分为程序计数器、虚拟机栈、本地方法栈、Java堆、方法区（运行时常量池）、直接内存。

我们都知道Java虚拟机是多线程的，在运行时数据区中，虚拟机栈、本地方法栈、程序计数器属于线程私有的，每一个线程，就会有一份这三块区域。而方法区和堆则属于线程共享的。
##### 程序计数器
- 指向当前线程正在执行的字节码指令的地址

为什么需要程序计数器呢？我们都知道操作系统为我们提供了一种CPU时间片轮转机制，而我们的JVM实际上是运行在操作系统上的一层软件，因此就有可能被切换出CPU，或者被挂起。这时候我们就需要一个程序计数器去确保JVM的正常执行。程序计数器是JVM内存区域中唯一不会发生OOM的存在。

来看这么一个例子。
```java
// Person.java
public class Person {

    public  int work()throws Exception{
        //运行过程中 打包一个栈帧
        int x =1;//x是一个局部变量
        int y =2;
        int z =(x+y)*10;
        return  z;
    }
    public static void main(String[] args) throws Exception{
        Person person = new Person();//person 一个引用， new Person(）对象
        person.work();
    }
}
```
通过javap反编译，可以得到汇编代码
```java
public class com.jvm.ex1.Person {
  public com.jvm.ex1.Person();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public int work() throws java.lang.Exception;
    Code:
       0: iconst_1        //将int型1入操作数栈
       1: istore_1        //将操作数栈中栈顶的int型数值，存入局部变量表（下标为1的地方）
       2: iconst_2        //将int型2入操作数栈
       3: istore_2        //将操作数栈中栈顶的int型数值，存入局部变量表（下标为2的地方）
       4: iload_1         //将局部变量表中下标为1的int型数据入栈
       5: iload_2         //将局部变量表中下标为2的int型数据入栈
       6: iadd            //1)将栈顶两int型数值出栈 2)相加 3)将结果压入操作数栈
       7: bipush  10      //10的数值扩展成int值压入操作数栈
       9: imul            //1)将栈顶两int型数值出栈 2)相乘 3)将结果压入操作数栈
      10: istore_3        //将操作数栈中栈顶的int型数值，存入局部变量表（下标为3的地方）
      11: iload_3         //将局部变量表中下标为3的int型数据入栈
      12: ireturn         //退出方法

  public static void main(java.lang.String[]) throws java.lang.Exception;
    Code:
       0: new           #2                  // class com/jvm/ex1/Person
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: invokevirtual #4                  // Method work:()I
      12: pop
      13: aload_1
      14: invokevirtual #5                  // Method java/lang/Object.hashCode:()I
      17: pop
      18: bipush        12
      20: istore_2
      21: return
}
```
在Code区域有0，1，2，3……的数字，这就是针对方法体的偏移量，可以大体认为等于程序计数器，记录字节码的地址。不同指令的偏移量是不一样的。

##### 虚拟机栈
- 存储当前线程运行方法所需的数据，指令、返回地址

栈的特性我们都是知道，是一种先进后出的数据结构，而虚拟机栈的数据我们称为栈帧，栈帧中又包含了多个数据。
- 栈帧
  - 局部变量表　　　用于存储局部变量
  - 操作数栈　　　　存放方法的操作
  - 动态连接　　　　因为 Java 是在运行期间动态链接的，所以为了支持动态链接，需要将方法区里面的符号引用转为直接引用（即：给出地址），这就叫动态链接
  - 返回地址　　　　方法结束时返回的地址

我们的线程执行方法的时候，每调用一个方法就会在虚拟机栈中入栈一个栈帧，但是虚拟机栈的大小是有限制的（-Xss），所以方法不能无限的调用。

在上面的汇编代码中，可以看到``work()``方法执行时候的的操作。
##### 本地方法栈
- 本地方法栈保存的是native方法的信息

我们知道，Java的解释执行是通过栈的，当一个JVM创建的线程调用native方法后，JVM不再为其在虚拟机栈中创建栈帧，JVM只是简单地动态链接并直接调用native方法。

##### Java堆
存放对象实例和数组

Java堆的大小参数设置：
- -Xmx  堆区内存可被分配的最大上限
- -Xms  堆区内存初始内存分配的大小
##### 方法区
方法区又叫做运行时常量池，里面会存放类信息，常量，静态变量和即时编译期编译后的代码。

既然Java堆和方法区中的对象都是线程共享的，那么为什么不只划分为一个区域呢？因为Java堆中存放的对象实例和数组，是频繁的发生GC回收的，而方法区中的对象是相对稳定的。因此为了GC回收的高效，便划分为Java堆和方法区。

##### 直接内存
直接内存不是虚拟机运行时数据区的一部分，也不是java虚拟机规范中定义的内存区域；
- 如果使用了NIO,这块区域会被频繁使用，在java堆内可以用directByteBuffer对象直接引用并操作；
- 这块内存不受java堆大小限制，但受本机总内存的限制，可以通过MaxDirectMemorySize来设置（默认与堆内存最大值一样），所以也会出现OOM异常；

##### HSDB
>HSDB是jdk为我们提供的一个可视化的可监控JVM运行情况的工具。可以方便我们从底层去理解JVM
打开HSDB，监控JVM的运行情况。sa-jdi.jar在我们JDK的lib目录中
```c
java -cp ./sa-jdi.jar sun.jvm.hotspot.HSDB
```
jps 类似于Linux上的ps命令，可以显示操作系统中运行的Java进程

#### 深入辨析堆和栈
**功能**
以栈帧的方式存储方法调用的过程，并存储方法调用过程中基本数据类型的变量（int、short、long、byte、float、double、boolean、char等）以及对象的引用变量，其内存分配在栈上，变量出了作用域就会自动释放；

而堆内存用来存储Java中的对象。无论是成员变量，局部变量，还是类变量，它们指向的对象都存储在堆内存中；

**线程独享还是共享**
栈内存归属于单个线程，每个线程都会有一个栈内存，其存储的变量只能在其所属线程中可见，即栈内存可以理解成线程的私有内存。
堆内存中的对象对所有线程可见。堆内存中的对象可以被所有线程访问。

**空间大小**
栈的内存要远远小于堆内存，栈的深度是有限制的，可能发生StackOverFlowError问题。

#### 内存溢出
- 栈溢出
- 堆溢出
- 方法区溢出
- 本机直接内存溢出


#### 虚拟机优化技术
##### 编译优化技术
方法内联：
下面这个例子，main方法在调用max方法的时候，参数都已经确定了，其实就可以直接去判断a > b。
```java
public static void main(String[] args) {
   max(1, 2);//调用max方法：  虚拟机栈 --入栈（max 栈帧）
    // boolean i1 = 1>2;
}

public static boolean max(int a,int b){//方法的执行入栈帧。
    return a > b;
}
```
如果调用max方法的话，就会涉及到虚拟机栈中max方法栈帧的入栈和出栈。而方法内联就是在编译过程中将目标方法的方法体复制到调用方法中，带来性能的提升。


##### 栈的优化技术
**栈帧之间数据共享**
下面这个例子，我们通过HSDB可以观察到，work()方法栈帧的局部变量表会跟main()方法栈帧的操作数栈会有一部分共享。
```java
public class JVMStack {

    public int work(int x) throws Exception{
        int z =(x+5)*10;//局部变量表有
        Thread.sleep(Integer.MAX_VALUE);
        return  z;
    }
    public static void main(String[] args)throws Exception {
        JVMStack jvmStack = new JVMStack();
        jvmStack.work(10);//10  放入main栈帧操作数栈
    }
}
```
