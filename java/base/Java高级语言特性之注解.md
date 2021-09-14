# Java语言特性之注解
## 注解的定义
>Java 注解（Annotation）又称 Java 标注，是 JDK1.5 引入的一种注释机制。

注解是元数据的一种形式，提供有关于程序但不属于程序本身的数据。注解对它们注解的代码的操作没有直接影响。
注解本身没有任何意义，单独的注解就是一种注释，他需要结合其他如反射、插桩等技术才有意义。

## 如何定义一个注解
```Java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.SOURCE)
public @interface Example {
    String value() default "xxx";
}
```
这里是注解的一个简单的例子，在接口前面加上一个@，就能定义一个注解了。
在这个注解中还有一个value()的成员变量，其中我们为它赋了默认值“xxx”，如果没有默认值，那么该注解使用的时候，就必须为它传值。

注解上面还有两个注解，我们将之称为元注解。
## 元注解
>元注解，即在定义注解时，注解类也能够使用其他的注解声明。这种对注解类型进行注解的注解类，我们称之为 meta-annotation（元注解）。

声明的注解允许作用于哪些节点使用@Target声明，例如ElementType.FIELD允许在成员变量上使用，而@Target注解是一个一对多的关系，即我们所写的注解可以在多个地方定义，在类上定义，在方法上定义，等等；

保留级别由@Retention 声明。其中保留级别如下。
- **RetentionPolicy.SOURCE**
 - 标记的注解仅保留在源级别中，并被编译器忽略。
- **RetentionPolicy.CLASS**
 - 标记的注解在编译时由编译器保留，但 Java 虚拟机(JVM)会忽略。
- **RetentionPolicy.RUNTIME**
 - 标记的注解由 JVM 保留，因此运行时环境可以使用它。

当我们使用@SOURCE声明注解时候，@Example注解的保留级别为SOURSE，即保留到源码阶段。
```Java
@Example("123")
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }
}
```
通过ASM反编译工具查看MainActivity.class字节码，并没有看到注解的存在，因为在编译过程中，注解已经被抹除了
```Java
// class version 51.0 (51)
// access flags 0x21
这里并没有看到注解的存在了
public class com/example/anatationtest/MainActivity extends androidx/appcompat/app/AppCompatActivity {

  // compiled from: MainActivity.java

  // access flags 0x1
  public <init>()V
  ......
  protected onCreate(Landroid/os/Bundle;)V
   ......
}
```
## 注解的应用场景
根据注解的保留级别不同，对注解的使用自然存在不同场景。由注解的三个不同保留级别可知，注解作用于：
源码、字节码与运行时可以产生不同的应用场景。

级别|技术|说明|
:-:|:-:|-
源码|APT|在编译期能够获取注解与注解声明的类包括类中所有成员信息，一般用于生成额外的辅助类|
字节码|字节码增强|在编译出Class后，通过修改Class数据以实现修改代码逻辑目的。对于是否需要修改的区分或者修改为不同逻辑的判断可以使用注解|
运行时|反射|在程序运行期间，通过反射技术动态获取注解与其元素，从而完成不同的逻辑判定|

### APT技术
>APT技术，APT，全称为Annotation Processor Tools ，即注解处理器

要定义一个注解处理器，首先要新建一个complier模块，并在app模块中依赖。
在其中新建一个注解处理器，继承自AbstractProcessor，编译器已经为我们在内部实现了注解的采集，我们只需要对注解进行处理就可以了。
```java
@SupportedAnnotationTypes("com.example.anatationtest.Example")
public class ExampleProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        Messager messager = processingEnv.getMessager();
        messager.printMessage(Diagnostic.Kind.NOTE,"========这里是打印信息========");
        return false;
    }
}
```
正如Activity类需要在Manifest中注册一样，注解器也需要配置才可以生效，就是在complier模块的resources目录下新建META-INF/services，然后新建File，并重命名为`javax.annotation.processing.Processor`

其中配置文件内容为，即定义的注解处理器的路径：
```java
com.example.compile.ExampleProcessor
```
#### 注解处理程序运行在什么时候
我们知道，一个.java文件要由javac编译成.class文件，并交由虚拟机去运行。在这个过程中，javac会采集到所有的注解信息，并包装成Element节点，然后交给注解处理程序。那么怎么证明这一点呢？

试着Make Project，可以在Build Output中的compileDebugJavaWithJavac Task中看到我们写在代码中的打印信息，说明javac编译.java文件的阶段调起了注解处理程序。

#### Android注解语法检查
在Android中我们需要设计接口以供使用者调用时，如出现需要对入参进行类型限定，如限定为资源ID、布局ID等类型参数，将参数类型直接给定int即可。然而，我们可以利用Android为我们提供的语法检查注解，来辅助进行更为直接的参数类型检查与提示。

如参数限制为：图片资源ID。这里利用了***@Drawable*** 来限定入参为Drawable类型的int值
```java
public Drawable getMyDrawable(@DrawableRes int id) {
       return getDrawable(id);
}
```
在平时开发中假如有一个方法限制了入参的类型，那么我们可以使用枚举来解决。
```Java
private Weekday currentDay;

    enum Weekday {
        SUNDAY,MONDAY
    }
    public void setCurrentDay(Weekday currentDay){
        this.currentDay=currentDay;
}
```
但是通过ASM字节码工具可以发现，枚举其实是生成了对象，较int基本数据类型会比较占用内存
```java
// access flags 0x4019
public final static enum Lcom/enjoy/ annotat ion/ intdef/Test$WeekDay; SUNDAY
// access flags 0x4019
public final static enum Lcom/ enjoy/ annotation/ intdef /Test$WeekDay; MONDAY

```

这时候就可以使用@IntDef注解来进行语法检查，@IntDef是AndroidX为我们提供的一个元注解。由IDE来实现，在我们编写代码的时候进行检查。

```java
@WekDay
private static int mCurrentIntDay;

@IntDef({SUNDAY, MONDAY})
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.SOURCE)
@interface WekDay {  //注解

}

public static void setCurrentDay(@WekDay int currentDay) {
   mCurrentIntDay = currentDay;
}
```
但是，语法检查阶段对于我们编译是不会产生影响的。
### 字节码增强技术
>什么叫字节码增强技术？就是在字节码中写代码。平时我们是在.java文件中去编写代码，其有一定的格式，也正如.java一样，.class文件也有一定的格式（数据按照特定的方式记录与排列）。

QQ空间曾经发布的热修复解决方案中利用Javaassist 库实现向类的构造函数中插入一段代码解决
CLASS_ISPREVERIFIED 问题。包括了Instant Run的实现以及参照Instant Run实现的热修复美团Robus等等等等都利用到了插桩技术。

插桩就是将一段代码插入到另一段代码，或替换另一段代码。字节码插桩顾名思义就是在我们编写的源码编译成字节码（Class）后，在Android下生成dex之前修改Class文件，修改或者增强原有代码逻辑的操作。

由于对于该技术也只是有所了解，因此不做更多赘述。如果大家感兴趣可以自己去加强学习。

## 利用注解加反射实现findViewById
我们利用注解加反射来实现一个简单的findViewById。

首先我们定义一个` @InjectView`的注解 ，里面包含一个``@Idres int ``类型的成员变量，用来存放控件的id值。

由于程序需要在运行期间利用反射来获取元素的注解和值，因此注解应声明在Runtime阶段执行：
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface InjectView {
    @IdRes int value();
}
```
使用注解：
```java
public class MainActivity extends AppCompatActivity {

    @InjectView(R.id.tv_text)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        InjectUtil.injectView(this);
        textView.setText("使用了@InjectView注解");
    }
}
```
**InjectViewUtil 处理工具类**
```java
public class InjectUtil {
    public static void injectView(Activity activity) {
        Class<? extends Activity> cls = activity.getClass();
        //获得成员变量
        Field[] declaredFields = cls.getDeclaredFields();
        for (Field declaredField : declaredFields) {
            //判断是否被@Inject注解
            if (declaredField.isAnnotationPresent(InjectView.class)) {
                InjectView annotation = declaredField.getAnnotation(InjectView.class);
                //获得注解的值
                int id = annotation.value();
                View view = activity.findViewById(id);
                //反射设置属性的值
                declaredField.setAccessible(true);//设置访问权限，允许操作private属性
                try {
                    declaredField.set(activity, view);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

InjectViewUtil在处理时，先通过getClass拿到Activity的类对象，再使用``getDeclaredFields``拿到其成员变量，并通过``if (declaredField.isAnnotationPresent(InjectView.class))``
判断是否被@InjectView注解过了，并进一步拿到id值，最后使用set方法设置回去。运行项目，观察效果，TextView 对象已经获取到了实例，并修改为了我们设置的text值。

这是ButterKnife早期的实现，但由于运行阶段利用反射去处理注解，会影响运行时的性能，所以后面它是在编译时对注解进行解析完成相关代码的生成，即刚刚介绍的第一种，利用注解处理器去完成findViewById的过程，相关源码大家感兴趣的话可以去查阅。

关于java注解的知识本次就介绍到这~

**じゃ、また**
