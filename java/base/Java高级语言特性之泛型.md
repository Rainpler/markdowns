# Java高级语言特性之泛型
>Java泛型（generics）是JDK 5中引入的一个新特性，泛型提供了编译时类型安全监测机制，该机制允许程序员在编译时监测非法的类型。使用泛型机制编写的程序代码要比那些杂乱地使用Object变量，然后再进行强制类型转换的代码具有更好的安全性和可读性。

泛型在java中有很重要的地位，在面向对象编程及各种设计模式中有非常广泛的应用。

为了方便介绍，本文从以下几个方面来展开：
1. 为什么我们需要泛型
2. 泛型类、泛型接口、泛型方法
3. 如何限定类型变量
4. 泛型使用中的约束和局限性
5. 泛型类型能继承吗？
6. 泛型中通配符类型
7. 虚拟机是如何实现泛型的？

## 为什么我们需要泛型
首先我们先来看两行代码，就知道为什么要使用泛型了。
```Java
public class NonGeneric {

    public int addInt(int x,int y){
        return x+y;
    }

    public float addFloat(float x,float y){
        return x+y;
    }

    public static void main(String[] args) {
        NonGeneric nonGeneric = new NonGeneric();
        System.out.println(nonGeneric.addInt(1,2));
        System.out.println(nonGeneric.addFloat(1f,2f));
    }
}
```
在这里，我们打算定义这么一个方法，完成数值类型求和，为了计算int型的和，我们定义了``addInt()``方法，为了计算float型的和，我们又定义了``addFloat()``方法，那么还想计算两个double类型的和呢，是不是又要定义一个``addDouble()``。
再看一个例子：
```java
public class NonGeneric2 {
    public static void main(String[] args) {
        List list = new ArrayList();
        list.add("mark");
        list.add("OK");
        list.add(100);
        for (int i = 0; i < list.size(); i++) {
            String name = (String) list.get(i); // 1
            System.out.println("name:" + name);
        }
    }
}
```
定义了一个``List``类型的集合，先向其中加入了两个字符串类型的值，随后加入一个``Integer``类型的值。这是完全允许的，因为此时list默认的类型为``Object``类型。在之后的循环中，由于忘记了之前在list中也加入了``Integer``类型的值或其他编码原因，很容易出现类似于//1中的错误。因为编译阶段正常，而运行时会出现``java.lang.ClassCastException``异常。因此，导致此类错误编码过程中不易发现。

为了解决这个问题，我们可以在定义阶段，为List添加泛型``List<String>list = new ArrayList();``,这样一来，在编码过程中，就会进行类型检查，编译阶段就可以发现错位。
 在如上的编码过程中，我们发现主要存在两个问题：
1. 当我们将一个对象放入集合中，集合不会记住此对象的类型，当再次从集合中取出此对象时，改对象的编译类型变成了Object类型，但其运行时类型任然为其本身类型。
2. 因此，//1处取出集合元素时需要人为的强制类型转化到具体的目标类型，且很容易出现``java.lang.ClassCastException``异常。

**所以泛型的好处就是：**
- 	适用于多种数据类型执行相同的代码
- 	泛型中的类型在使用时指定，不需要强制类型转换

## 泛型类、泛型接口、泛型方法

泛型，即“参数化类型”。一提到参数，最熟悉的就是定义方法时有形参，然后调用此方法时传递实参。那么参数化类型怎么理解呢？
顾名思义，就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式（可以称之为类型形参），然后在使用/调用时传入具体的类型（类型实参）。

泛型的本质是为了参数化类型（在不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型）。也就是说在泛型使用过程中，操作的数据类型被指定为一个参数，这种参数类型可以用在类、接口和方法中，分别被称为泛型类、泛型接口、泛型方法。

#### 泛型类
那么如何来定义一个泛型类呢，引入一个类型变量T（其他大写字母都可以，不过常用的就是T，E，K，V等等），并且用<>括起来，并放在类名的后面。泛型类是允许有多个类型变量的。
```java
public class NormalGeneric<T> {
    private T data;

    public NormalGeneric() {
    }

    public NormalGeneric(T data) {
        this.data = data;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```
在使用的时候，new 出一个``NormalGeneric``实例，并传入泛型类型为``String``。如果我们``setData(1)``的时候，就会报参数类型不匹配。如果我们不为其指定泛型类型，则等价于泛型为``Object``
```java
public static void main(String[] args) {
      NormalGeneric<String> normalGeneric = new NormalGeneric<>();
      normalGeneric.setData("OK");
      //normalGeneric.setData(1);
      System.out.println(normalGeneric.getData());
      NormalGeneric normalGeneric1 = new NormalGeneric();
      normalGeneric1.setData(1);
      normalGeneric1.setData("dsf");
  }
```
#### 泛型接口
泛型接口与泛型类的定义基本相同，同样是在后面引入一个类型变量T，并且用<>括起来。
```java
public interface Genertor<T> {
    public T next();
}
```
我们使用接口的时候，总要使用它的实现类，那么对于泛型接口，我们该怎么使用呢？
```java
//不指定泛型参数类型，实现为泛型类
public class ImplGenertor<T> implements Genertor<T> {
    @Override
    public T next() {
        return null;
    }
}
//指定泛型参数类型
public class ImplGenertor2 implements Genertor<String> {
    @Override
    public String next() {
        return null;
    }
}
```
我们可以在实现泛型接口的时候，不指定泛型的类型，实现为泛型类。此外，还可以在实现泛型接口的时候，指定泛型的类型。

#### 泛型方法
是在调用方法的时候指明泛型的具体类型 ，泛型方法可以在任何地方和任何场景中使用，包括普通类和泛型类。注意泛型类中定义的普通方法和泛型方法的区别。

**普通方法：**
```java
//定义一个泛型类，泛型形参为T
public class Generic<T>{
    private T key;
    public Generic(T key) {
        this.key = key;
    }

    //这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
    //所以在这个方法中才可以继续使用 T 这个泛型。
    public T getKey(){
          return key;
      }

    /**
     * 这个方法显然是有问题的，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"
     * 因为在类的声明中并未声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别。
     */
//        public E setKey(E key){
//            this.key = key;
//        }
}
```
我们定义了一个泛型类``Generic<T>``，声明泛型形参为``T``，内部有一个``getKey()``的方法，虽然这个方法中使用了泛型，但这并不是泛型方法，因为它的返回值参数是在实例化泛型类时，所传入泛型参数的实际类型，因此它只是一个普通的成员方法。

如果我们未在类的声明中声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"。

**泛型方法：**
```java
public class GenericMethod<T> {

    //首先在public与返回值之间的<T>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T
    //这个T可以出现在这个泛型方法的任意位置.
    //泛型的数量也可以为任意多个
    public <T> T genericMethod(T...a){
        return a[a.length/2];
    }

    public static void main(String[] args) {
        GenericMethod genericMethod = new GenericMethod();
        System.out.println(genericMethod.<String>genericMethod("mark","av","lance"));
        System.out.println(genericMethod.genericMethod(12,34));
    }
}
```
定义泛型方法的时候，首先在public与返回值之间的\<T\>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T，这个T可以出现在这个泛型方法的任意位置，泛型的数量也可以为任意多个。要注意的是，在这里泛型类的T与泛型方法的T不一定是同一种类型，因为泛型方法中的泛型是一种全新的类型。
## 如何限定类型变量
有时候，我们需要对类型变量加以约束，比如计算两个变量的最小，最大值。
```Java
public static <T> T min(T a,T b){
    if(a.compareTo(b)>0) return a; else return b;
}
```
如何确保传入的两个变量一定有compareTo方法？那么解决这个问题的方案就是将T限制为实现了接口Comparable的类。
```java
public static <T extends Comparable> T min(T a,T b){
        if(a.compareTo(b)>0) return a; else return b;
    }
```
T表示应该绑定类型的子类型，Comparable表示绑定类型，子类型和绑定类型可以是类也可以是接口。如果这个时候，我们试图传入一个没有实现接口Comparable的类的实例，将会发生编译错误。

同时extends左右都允许有多个，如`` <T,V extends Comparable & Serializable>``。注意限定类型中，只允许有一个类，而且如果有类，这个类必须是限定列表的第一个。这种类的限定既可以用在泛型方法上也可以用在泛型类上。

## 泛型使用中的约束和局限性
```java
public class Restrict<T> {
    private T data;

    //不能实例化类型变量
    public Restrict() {
        this.data = new T();
    }

    //静态域或者方法里不能引用类型变量
    //private static T instance;

    //静态方法 本身是泛型方法就行
    private static <T> T getInstance(){}

    public static void main(String[] args) {
        //Restrict<double> 不能用基本类型实例化类型参数
        Restrict<Double> restrict = new Restrict<>();
        //运行时类型查询只适用于原始类型
//        if(restrict instanceof  Restrict<Double>) 不允许
//        if(restrict instanceof  Restrict<T>) 不允许

        Restrict<String> restrictString= new Restrict<>();


        //编译的时候会泛型擦除
        System.out.println(restrict.getClass()==restrictString.getClass()); //true
        System.out.println(restrict.getClass().getName()); //Restrict
        System.out.println(restrictString.getClass().getName()); //Restrict

        //可以定义泛型数组，但不能实例化
        Restrict<Double>[] restrictArray;
        //Restrict<Double>[] restricts = new Restrict<Double>[10];
        //ArrayList<String>[] list1 = new ArrayList<String>[10];
        //ArrayList<String>[] list2 = new ArrayList[10];

    }
}
```
上述代码中的例子，都是错误的泛型使用方式。

不能在静态域或方法中引用类型变量。因为泛型是要在对象创建的时候才知道是什么类型的，而对象创建的代码执行先后顺序是static的部分，然后才是构造函数等等。所以在对象初始化之前static的部分已经执行了，如果你在静态部分引用的泛型，那么毫无疑问虚拟机根本不知道是什么东西，因为这个时候类还没有初始化。

再来看下面的代码段，泛型类不可以继承自``Exception/Throwable``，在错误处理时，不能捕获泛型类对象，但是可以抛出泛型异常
```java
public class ExceptionRestrict {

    /*泛型类不能extends Exception/Throwable*/
    //private class Problem<T> extends Exception;

    /*不能捕获泛型类对象*/
//    public <T extends Throwable> void doWork(T x){
//        try{
//
//        }catch(T x){
//            //do sth;
//        }
//    }

    public <T extends Throwable> void doWorkSuccess(T x) throws T{
        try{

        }catch(Throwable e){
            throw x;
        }
    }
}

```
## 泛型类型能继承吗？
我们先来定义这样两个类，他们之间为继承关系
```java
public class Fruit {
}

public class Apple extends Fruit {
}
```
然后我们来看下面的代码段
```Java
public class Pair<T> {
    public static void main(String[] args) {
        Pair<Fruit> fruit = new Pair<>();
        Pair<Apple> apple = new Pair<>();
        //1. Pair<Apple>apple2  = new Pair<Fruit>();  参数类型不匹配
    }
  }
```
``Pair``类为一个泛型类，分别定义泛型类型为``Apple``和``Fruit``的两个Pair对象，那么这两个对象之间具有继承关系吗？答案是没有任何关系，1. 处的代码编译器会报参数类型不匹配。但是泛型类可以继承或者扩展其他泛型类，比如最常用的``List``和``ArrayList``。
```java
private static class ExtendPair<T> extends Pair<T>{}
```
## 泛型中通配符类型
正是因为前面所述的，Pair<Employee>和Pair<Worker>没有任何关系，如果我们有这样的代码段，有那么一个泛型方法set，期望传入的参数是一个``Pair<Apple>``类型对象，set方法将不可以传入``Pair<Fruit>``对象。
```java
private static <T> void set(Pair<Apple> p){
}
public static void main(String[] args) {
       Pair<Apple> apple = new Pair<>();
       Pair<Fruit> fruit = new Pair<>();
       set(apple);
       //set(fruit); 参数类型错误
   }
```
但是，我们又想实现这样的需求，既然苹果是水果，那么为什么不能传入水果呢？希望set方法也能传入``Pair<Fruit>``对象，也就是原本类的继承关系在泛型中还能继续实现，这就有了通配符的诞生。为了完成上面的需求，我们可以将set方法改为：
```java
private static <T> void set(Pair<? extends Fruit> p){
}
```
这里的``?``就是通配符，但是通配符只能用在方法上，而不能用于类的定义上。比如这里会报``unexpected wildchar``，不期望的通配符。
```java
public class Apple <? extends Fruit> {
}
```
那么这里的``<? extends Fruit>`` extends表示什么呢？这里限定了类型的上限，即继承的类型上界为``Fruit``,假如``Fruit``还继承自``Plant``,set方法则不能传入``Pair<Plant>``的对象参数。

**通配符的限制**

那么既然通配符解决了上述的问题，那么它是否存在什么问题？
```java
public class GenericType<T> {
    private T data;

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}

public static void main(String[] args){
      GenericType<? extends Fruit> c =  new GenericType<>();

      Apple apple =  new Apple();
      Fruit fruit = new Fruit();
      //c.setData(apple);
      //c.setData(fruit);
      Fruit x = c.getData();
      //Apple x = c.getData();
  }
```
在这里，``setData()``方法传入Apple和Fruit对象，都会报错，因为对于``<? extends Fruit>``这样的通配符泛型来说，编译器只知道它是``Fruit``的子类，但是并不知道是哪个子类，所以getData()赋值给``Fruit x``是可以的，而赋值给``Apple x``是不行的。**因此extends是用于安全地访问数据**。

那么正因为``setData()``方法存在这样的问题，所以又提出了``<? super Apple>``这样的使用方式，这里限定了传入的参数为``Apple``的类本身或者超类，即限定类型的下界。
```java
public static void printSuper(GenericType<? super Apple> p){
      System.out.println(p.getData());
  }
public static void main(String[] args){
       GenericType<Fruit> fruitGenericType = new GenericType<>();
       GenericType<Apple> appleGenericType = new GenericType<>();
       GenericType<HongFuShi> hongFuShiGenericType = new GenericType<>(); //HongFuShi为Apple的子类
       GenericType<Orange> orangeGenericType = new GenericType<>();  //Orange为Fruit的子类，与Apple平行

       printSuper(fruitGenericType);
       printSuper(appleGenericType);
//        printSuper(hongFuShiGenericType);
//        printSuper(orangeGenericType);

       //表示GenericType的类型参数的下界是Apple
       GenericType<? super Apple> x = new GenericType<>();
       x.setData(new Apple());
       x.setData(new HongFuShi());
       //x.setData(new Fruit());
       Object data = x.getData();//返回类型只能是Object
   }
```
如果我们泛型使用``super``，这里的``setData()``方法只能传入``Apple``或者它的子类，而不能传入它的父类，因为``Apple``和``HongFuShi``都可以安全转型为``Apple``，而``Fruit``不可以。而``getData()``返回的一定是``Apple``的超类，但是是哪个超类，编译器不知道，但是对于Java来说，所有类的超类都是``Object``，因此可以安全地向上转型。**因此super是用于安全地写入数据**。

无限定的通配符 ?，表示对类型没有什么限制，可以把？看成所有类型的父类，如``Pair< ?>``；
比如：
``ArrayList<T> al=new ArrayList<T>()``; 指定集合元素只能是T类型
``ArrayList<?> al=new ArrayList<?>()``;集合元素可以是任意类型，这种没有意义，一般是方法中，只是为了说明用法。


## 虚拟机是如何实现泛型的？

泛型思想早在C++语言的模板（Template）中就开始生根发芽，在Java语言处于还没有出现泛型的版本时，只能通过Object是所有类型的父类和类型强制转换两个特点的配合来实现类型泛化。，由于Java语言里面所有的类型都继承于java.lang.Object，所以Object转型成任何对象都是有可能的。但是也因为有无限的可能性，就只有程序员和运行期的虚拟机才知道这个Object到底是个什么类型的对象。在编译期间，编译器无法检查这个Object的强制转型是否成功，如果仅仅依赖程序员去保障这项操作的正确性，许多``ClassCastException``的风险就会转嫁到程序运行期之中。

泛型技术在C#和Java之中的使用方式看似相同，但实现上却有着根本性的分歧，C#里面泛型无论在程序源码中、编译后的IL中（Intermediate Language，中间语言，这时候泛型是一个占位符），或是运行期的CLR中，都是切实存在的，``List<int>``与``List<String>``就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为类型膨胀，基于这种方法实现的泛型称为真实泛型。

Java语言中的泛型则不一样，它只在程序源码中存在，在编译后的字节码文件中，就已经替换为原来的原生类型（Raw Type，也称为裸类型）了，并且在相应的地方插入了强制转型代码，因此，对于运行期的Java语言来说，``List<int>``与``List<String>``就是同一个类，所以泛型技术实际上是Java语言的一颗语法糖，Java语言中的泛型实现方法称为**类型擦除**，基于这种方法实现的泛型称为伪泛型。

来看下面的代码段：
```java
public class Theory {
    public static void main(String[] args) {
        Map<String,Integer> map = new HashMap<>();
        map.put("mark",18);
        System.out.println(map.get("mark"));
    }
}
```
反编译过后的class文件为，在``(String)map.get()``处添加了强制转型代码
```java
public class Theory {
    public Theory() {
    }

    public static void main(String[] args) {
        Map<String, String> map = new HashMap();
        map.put("mark", "18");
        System.out.println((String)map.get("mark"));
    }
}
```
下面这段代码是不能被编译的，因为参数``List<Integer>``和``List<String>``编译之后都被擦除了，变成了一样的原生类型``List<E>``，擦除动作导致这两种方法的特征签名变得一模一样。
```java
public static String method(List<String> stringList){
    System.out.println("List");
    return "OK";
}

 public static Integer method(List<Integer> stringList){
     System.out.println("List");
     return 1;
 }
```
由于Java泛型的引入，各种场景（虚拟机解析、反射等）下的方法调用都可能对原有的基础产生影响和新的需求，如在泛型类中如何获取传入的参数化类型等。因此，JCP组织对虚拟机规范做出了相应的修改，引入了诸如Signature、LocalVariableTypeTable等新的属性用于解决伴随泛型而来的参数类型的识别问题。

Signature(弱记忆)是其中最重要的一项属性，它的作用就是存储一个方法在字节码层面的特征签名，这个属性中保存的参数类型并不是原生类型，而是包括了参数化类型的信息。修改后的虚拟机规范要求所有能识别49.0以上版本的Class文件的虚拟机都要能正确地识别Signature参数。

另外，从Signature属性的出现我们还可以得出结论，擦除法所谓的擦除，仅仅是对方法的Code属性中的字节码进行擦除，实际上元数据中还是保留了泛型信息，这也是我们能通过反射手段取得参数化类型的根本依据。
