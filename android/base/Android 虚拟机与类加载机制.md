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

## ClassLoader
我们都知道编写的类是通过ClassLoader加载的，那么在Andoird中，Class类加载机制是怎样的呢。在java.lava包中，有一个ClassLoader的抽象类，这是所有类加载器的基类。

ClassLoader主要有两个子类，其中BootClassLoader用于加载Android Framework层class文件。而BaseDexClassLoader中又有DexPathList、PathClassLoader和DexClassLoader等三个主要的子类。
- DexPathList
- PathClassLoader  　 Android应用程序类加载器
- DexClassLoader  　  额外提供的动态类加载器

![](https://ae01.alicdn.com/kf/Ufac4f359564c4570a80af97966227514m.jpg)

## 热修复
