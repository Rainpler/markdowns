为什么要学习Kotlin？想必做Java开发的同学们，都基本接触过Intellij Idea这款大名鼎鼎的Java编程语言开发撰写时所用的集成开发环境吧。而这款ide，则是由业界知名的软件开发公司JetBrains打造的。而Kotlin正是由该公司开发的一个用于现代多平台应用的静态编程语言。Kotlin可以编译成Java字节码，支持在JVM上运行；也可以编译成JavaScript，方便在没有JVM的设备上运行。Kotlin现代，简洁，安全，已正式成为Android官方支持开发语言。

[kotlin官方文档](https://kotlinlang.org/)

作为一门全新的语言，我们必须从它的基础语法开始学习。下面将从5个部分来展开叙说。
- Kotlin基础语法
- Kotlin比较与数组
- Kotlin条件控制
- Kotlin循环与标签
- Kotlin类与对象

## Kotlin基础语法
#### 变量的定义
在Kotlin中，可以通过var和val来定义变量，不同的的是，前者是可变的，后者是不可变的。
- var <标识符> : <类型> = <初始化值>
- val <标识符> : <类型> = <初始化值>  有一点点类似Java中final修饰的变量

来看这么一个例子，有两个String变量，一个是使用var定义，一个使用val定义。
```kotlin

class Test {
    // 可以改，可以读  get  set
    var info1 : String = "A"

    // 只能读， 只有 get
    val info2 : String = "B"
}
```
导语已经提到过了，Kotlin可以编译成Java字节码，我们在AS中选择一个.kt文件，找到Tools->Kotlin->Show Kotlin Bytecode就可以得到其字节码，然后点击Decompile反编译为Java文件。
```kotlin
public final class Test {
   @NotNull
   private String info1 = "A";
   @NotNull
   private final String info2 = "B";

   @NotNull
   public final String getInfo1() {
      return this.info1;
   }

   public final void setInfo1(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.info1 = var1;
   }

   @NotNull
   public final String getInfo2() {
      return this.info2;
   }
}
```
我们观察该类，info2变量被final关键字修饰，是不是跟前面说到的不可修改是一致的，同时，为info1生成了get和set方法，为info2只生成了get方法。因此，var定义的变量具有可读写性，而val定义的变量则只具有可读性。

Kotlin还支持类型自动推导，而不需要我们自己指定，同时Kotlin是一种静态语言，在编译器就决定了变量的类型。
```kotlin
//类型推导
var info1 = "AAAA" // String
var info2 = 'A' //  Char
var info3 = 99 //  Int

var info4 = "LISI"  // info4==String类型
// info4 = 88   // 这么赋值会报错
```

#### 函数的定义
在Kotlin中，函数的定义类似JavaScript。在Java中，函数必须定义在Class下面一级，而在Kotlin中，函数的定义和类的定义可以是平级的。
- fun <方法名>(<参数类型> ：<标识符>) : <返回类型> {}
需要注意的是，void需要表示为Unit
```kotlin
fun main(): Unit {
   val a = add(1, 2)
}

fun add(number1: Int, number2: Int): Int {
    return number1 + number2
}
```
如同在变量定义那般，在函数定义中同样支持类型推导
```kotlin
// 返回类型  == 类型推导 Int
fun add2(number1: Int, number2: Int) = number1 + number2

// 返回类型  == 类型推导 String
fun add3(number1: Int, number2: Int) = "AAA"

```
在Java中，我们可以有这样(int...args)的可变参数，Kotlin中也可以使用可变参数
```kotlin
fun lenMethod(vararg value: Int) {
    for (i in value) {
        println(i)
    }
}
```
Kotlin还可以在方法中使用lambda表达式函数，其形式如下。
- val <方法名> : (<参数类型>) -> <返回类型> ={ <实际参数> -> <返回值> }
```kotlin
fun main(): Unit {
   val addMethod : (Int, Int) -> Int = { number1, number2 -> number1 + number2 }
   val r = addMethod(9, 9)
   println(r)
}
```

#### 字符串模版
在Java中，我们格式化输出字符串的时候，通常是使用String.format()方法
```kotlin
String name = "张三";
int age = 28;
char sex = 'M';
String info = "ABCDEFG";
String format="name:%s,  age:%d,  sex:%c  info:%s";
System.out.println(String.format(format, name,age,sex,info));
```
而在Kotlin中，为我们提供了新的字符串模版使用。

**格式化代码的时候可以这么用**
- \$表示一个变量名或者变量值 　
- \$varName 表示变量值  　
- \${varName.fun()} 表示变量的方法返回值

```kotlin
val name = "张三"
val age = 28
val sex = 'M'
val info = "ABCDEFG"
println("name:$name,  age:$age,  sex:$sex  info:$info")
```
但是，如果我们想要打印$99999.99的话，就会跟\$符号产生冲突，因此我们需要这么来定义
```
val price = "${'$'}99999.99"
```
在字符串赋值的时候，我们通常使用'\n'来表示换行符，在Kotlin中，为我们提供了下面的使用方法，包裹字符串的双引号为三个一组，我们在回车的时候则可以自动换行，不需要再去多写'\n'了。
```kotlin
"""
|AAAAAAAAAAA
|BBBBBBBBBBB
"""
```
但是这样会在前面带上空格，使用``.trimIndent()``方法可以去除前置空格。假如我们还想去掉每一行前面的'|'，还可以使用``.trimMargin("|")``也给去除。

#### NULL检查机制
在Kotlin中，可以在定义一个变量的时候声明可为空，但是这种空安全设计对于声明可为空的参数，在使用时要进行空判断处理，这里有两种处理方式，字段后加!!像Java一样抛出空异常，另一种字段后加?。
```kotlin
var info: String? = null
println(info.length) //由于info可能为null，因此.length会导致空指针，编译失败

println(info?.length)  // 第一种补救：? 如果info是null，就不执行 .length

println(info!!.length) // 第2种补救： !! 我自己负责info 不会为null ==  (不管null不null必须执行)

if (info != null)      // 第3种补救
  println(info.length)

```
在函数中，同样可以使用？允许返回Null值
```kotlin
fun testMethod(name: String) : Int? {
    if (name == "zs") {
        return 99999
    }
    return null
}
```
#### 区间
在Kotlin中，区间跟Python类似
```kotlin
// 1 到 9
    for (i in 1..9) {
        println(i)
    }

    // 不会输出
    for (i in 9..1) {
        println(i)
    }

    // 大 到 小
    for (i in 9 downTo 1) {
        println(i)
    }

    // 用区间做判断
    val value = 88
    if (value in 1..100) {
        println("包了 1 到 100")
    }

    // 步长指定
    for (i in 1..20 step 2) {
        // 1 3 5 7 ...
        println(i)
    }

    // 排除 最后元素
    for (i in 1 until 10) {
        println(i)
    }
```

## Kotlin比较与数组
#### 比较
在Java中，字符串值比较通常使用``equals()``，而地址比较使用“==”，但是在Kotlin中，两个使用方式是等价的。如果需要比较对象地址的话，需要使用三个等号"\=\=\="
```kotlin
fun main() {

    val name1: String = "张三"
    val name2: String = "张三"

    // --- 比较值本身
    // == 等价 Java的equals
    println(name1.equals(name2))
    println(name1 == name2)


    // ---  比较对象地址
    val test1:Int? =  10000
    val test2:Int? =  10000
    println(test1 === test2) // false
}
```
#### 数组
在Kotlin中，定义数组通常有两种方式，遍历的时候则同样是用区间
```kotlin
//第一种
val numbers:Array<Int> = arrayOf(1,2,3,4,5,6,7,8)

//第二种
val numbers2 = Array(10,  {value: Int -> (value + 200) })

for(number: numbers){
  println(number)
}
```
## Kotlin条件控制
#### if语句
在Kotlin中，if语句是有返回值的，因此我们可以这么用。在花括号内，我们可以进行一系列操作，但是记得最后需要返回值
```kotlin
val number1: Int = 9999999
val number2: Int = 8888888

// 表达式 比 大小 最大值
val maxValue = if (number1 > number2) {
  //TODO ...
  println("number1更大")
  number1
} else {
  //TODO ...
  println("number2更大")
  number2
}

println(maxValue)
```
#### when语句
在Java中，switch用于条件选择，而在Kotlin中，则要使用when语句
```kotlin
val number = 5
when(number) {
     1 -> println("一")
     2 -> println("二")
     3 -> println("三")
     4 -> println("四")
     5 -> println("五")
     else -> println("其他")
 }
```
但是switch局限于case只能做值判断，而when不同，还可以做区间判断
```kotlin
val number = 745
when(number) {
    in 1..100 -> println("1..100")
    in 200..500 -> println("200..500")
    else -> println("其他")
}
```
像if语句一样，when方法里面同样有返回值。
```kotlin
val number = 3
    val result = when (number) {
        1 -> {
            println("很开心")
            // TODO ....
            "今天是星期一"
            99
        }
        2 -> {
            println("很开心")
            // TODO ....
            "今天是星期二"
            88
        }
        3 -> {
            println("很开心")
            // TODO ....
            "今天是星期三"
            true
            100
        }
        else -> 99
    }
```
如果需要像switch一样多个case 并列，则可以这么写
```kotlin
when (8) {
    1, 2, 3, 4, 5, 6, 7 -> println("满足")
    else -> println("不满足")
}
```
## Kotlin循环与标签
什么是标签？可以把它看成是一种标识符，我们先来看下面的代码
```kotlin
tttt@ for (i in 1..20) {
        for (j in 1..20) {
            println("i:$i, j:$j")
            if (i == 5) {
                // break // j循环 给break
                break@tttt // i循环 给break
            }
        }

    }
```
这里有两层循环，在第二层循环中，如果i==5的话，则跳出所有循环，如果直接使用break，那只能跳出当前循环，在外层循环再加一层判断。在Kotlin中，可以在循环前面添加标签，在循环体中通过break@标签名，直接退出循环。
- <标签名>@   自定义标签

在class中，还有自定义标签
```kotlin
class Derry {

    private val i = "AAAA"
    fun show() {
        println(i)
        println(this.i)
        println(this@Derry.i)
    }
}
```
在Kotlin中，循环可以有多种遍历方式
```kotlin
var items  = listOf<String>("李四", "张三", "王五")
for (item in items) {
    println(item)
}

items.forEach {
    println(it)
}

for (index in items.indices) {
    println("下标：$index,  对应的值：${items[index]}")
}
```
## Kotlin类与对象
在Java中，一个类的声明至少需要class关键字，类名，花括号。而在Kotlin中，一个最简单的类则是class 类名就可以。Kotlin中的类默认都是public。
#### 构造方法
Kotlin的构造函数分为主构造器（primary constructor）和次级构造器（secondary constructor）。
##### Primary Constructor
写法一：class 类名 constructor(形参1, 形参2, 形参3){}
```kotlin
class Person constructor(username: String, age: Int){
	private val username: String
	private var age: Int

	init{
		this.username = username
		this.age = age
	}
}

```
这里需要注意几点：

- 关键字constructor：在Java中，构造方法名须和类名相同；而在Kotlin中，是通过constructor关键字来标明的，且对于Primary Constructor而言，它的位置是在类的首部（class header）而不是在类体中（class body）。
- 关键字init：init{}它被称作是初始化代码块（Initializer Block），它的作用是为了Primary Constructor服务的，由于Primary Constructor是放置在类的首部，是不能包含任何初始化执行语句的，这是语法规定的，那么这个时候就有了init的用武之地，我们可以把初始化执行语句放置在此处，为属性进行赋值。在Kotlin中不同于Java，成员变量是没有默认值的，所以必须进行初始化赋值。当然我们也可以使用``lateinit``进行懒加载

写法二：
当constructor关键字没有注解和可见性修饰符作用于它时，constructor关键字可以省略（当然，如果有这些修饰时，是不能够省略的，并且constructor关键字位于修饰符后面）。那么上面的代码就变成：
```kotlin
class Person (username: String, age: Int){
    private val username: String
    private var age: Int

    init{
    	this.username = username
    	this.age = age
    }
}

```
初始化执行语句不是必须放置在init块中，我们可以在定义属性时直接将主构造器中的形参赋值给它
```kotlin
class Person(username: String, age: Int){
    private val username: String = username
    private var age: Int = age
}
```
这种在构造器中声明形参，然后在属性定义进行赋值，这个过程实际上很繁琐，有没有更加简便的方法呢？当然有，我们可以直接在Primary Constructor中定义类的属性。
```kotlin
class Person(private val username: String, private var age: Int){}
```
看，是不是一次比一次简洁？实际上这就是Kotlin的一大特点。我们如果没有为其显式提供Primary Constructor，Kotlin编译器会默认为其生成一个无参主构造，这点和Java是一样的
##### secondary constructor
和Primary Constructor相比，很明显的一点，Secondary Constructor是定义在类体中，第二，Secondary Constructor可以有多个，而Primary Constructor只会有一个。
```java
class Student constructor(username: String, age: Int) {
    private val username: String = username
    private var age: Int = age
    private var address: String
    private var isMarried: Boolean
    init {
        this.address = "Beijing"
        this.isMarried = false
    }
    constructor(username: String, age: Int, address: String) :this(username, age) {
        this.address = address
    }
    constructor(username: String, age: Int, address: String, isMarried: Boolean) : this(username, age, address) {
        this.isMarried = isMarried
    }
}
```
可以看到，我们可以使用this关键字来调用自己的其他构造器，并且需要注意它的语法形式，次级构造器: this(参数列表), 可以使用super关键字来调用父类构造器，次级构造会直接或者间接调用主构造。

**总结**
- 主构造方法只能有一个
- init代码块可以有多个
- init代码块和成员变量都是主构造方法的一部分，都是自上而下初始化
- 次构造方法可以有多个，如果有主构造此时次构造方法必须直接或间接的调用主构造方法
- 可以只有次构造方法，没有主构造方法
- 主次构造方法都没有时，编译器生成默认无参的一个主构造方法
- 如果子类继承父类，且父类显示的定义了构造方法，那么子类必须也要显示的定义一个构造方法并调用父类的一个构造方法来初始化父类

#### 抽象类和接口
在Kotlin中，定义接口也是使用interface关键字，默认都是open的。
```java
interface Callback {
    fun callbackMethod() : Boolean
}
```
使用abstract定义抽象类，并用: interface实现接口，abstract默认也是open的，如果不使用抽象类，继承必须添加open关键字
```java
abstract class Person : Callback , Callback2 {

    abstract fun getLayoutID() : Int

    abstract fun initView()

}
```
抽象类的实现类
```kotlin
class Student : Person() {

   override fun getLayoutID(): Int = 888

  override fun initView() { }

  override fun callbackMethod(): Boolean  = false
}
```
#### 单例和数据类
##### data数据类
在Java开发中，如果我们要实现一个网络接口，就必须有对应的Bean类去接受响应数据，为此我们需要写如下一大段代码：
```java
class User{
  private String user;
  private String name;
  private int age;

  public String getUser() {
        return user;
    }

    public void setUser(String user) {
        this.user = user;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```
而在Kotlin中，为我们提供了data关键字，可以更加简洁的完成Bean类的定义。

```kotlin
data class User(val user: String, val name: String, val age: Int){}
```

使用的时候则跟Java类似。data数据类还有``copy()``方法，用于实现浅拷贝
```kotlin
val user = User("aaa", "ming", 15)
```
##### 单例
我们在开发中，有的实体类只需要实例化一次，为此我们会将它实现为单例模式，在Kotlin中，可以使用object关键字，方便的为我们完成。
```kotlin
object MyEngine {

    fun m() {
        println("我就只有一个实例")
    }
}
```
假如我们来实现一个自己的单例，使用静态内部类模式的话，那么该怎么实现呢?
```kotlin
class NetManager {

    // 只有一个实例
    object Holder {
        val instance = NetManager()
    }

    // 看不到 static  可以 派生操作
    companion object {
        // 全部都是  相当于 Java static
        fun getInstance() : NetManager = Holder.instance
    }

    fun show(name: String) {
        println("show:$name");
    }

}
```
在kotlin中是不能使用static操作符的，但是可以使用companion派生操作，它会跟随类的诞生而诞生，在其内部的方法实现可以看做是静态的。

假如要通过懒加载实现单例的话，在Kotlin中实现如下：
```kotlin
class NetManager2 {

    companion object {
        private var instance: NetManager2? = null
        // 返回值：允许你返回null
        fun getInstance(): NetManager2? {
            if (instance == null) {
                instance = NetManager2()
            }

            // 如果是null，也返回回去了
            return instance
            // 第二种补救： 我来负责 instance 肯定不为null
            // return instance!!
        }
    }

    fun show(name: String) {
        println("show:$name");
    }

}
```
## Kotlin与Java互相调用
#### Java调用Kotlin方法
假如我们有那么一个Kotlin类，因为允许函数写在方法外，所以这里写了两个``show()``方法。
```kotlin
class Utils {

    fun show(info: String) {
        println(info)
    }

}
// MyUtils.kt 写了一个show      MyUtilsKt
fun show(info: String) {
    println(info)
}
```
在Java中调用的时候，如果调用类的成员方法，则跟Java中使用一致，如果调用类外方法的话，则使用``classRt.xxx()``的形式，系统会生成一个UtilsKt的类。
```java
public static void main(String[] args) {

        UtilsKt.show("Derry1");

        new Utils().show("new Derry2");
    }
```
#### Kotlin调用Java方法
假如我们的Java类如下，提供了一个in的静态变量和一个``getString()``方法
```java
public class Test {

    public static String in = "INNNNNN";

    public String getString() {
        return null;
    }
}
```
在Kotlin中，因为in是区间关键字所以引用的时候需要加上引号，而调用Java方法则会有"!"提醒，因此我们在使用的时候则最好通过局部变量引用，并使用"?"确保NULL检查机制安全。
```kotlin
fun main(){
   println(Test.`in`)
   var str: String  = Test().string
    println(str.length)
}
```
