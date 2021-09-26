我们都知道在Java中，匿名内部类会持有外部类的引用，那么怎么验证这个问题呢？接下来我们通过一个例子来证明一下：

首先创建一个接口：
```java
public interface Callback{
   void onCall();
}
```
然后是main()函数：
```java
public class Test {
    private String mName;

    public static void main(String args[]) {
        Test test = new Test();
        test.setName("小王");
        test.test();
    }

    private void test() {
        Callback callback = new Callback() {
            @Override
            public void onCall() {
                System.out.println(getName());
            }
        };
        callback.onCall();
    }

    private void setName(String name) {
        this.mName = name;
    }

    private String getName() {
        return this.mName;
    }
}
```
我们定义了一个接口，并在`test()`函数中，实例化了CallBack接口的匿名内部类对象，并调用call()，而call()方法中，又直接调用了外部类Test的getName()方法。

运行代码，观察打印结果：

```java
小王
```
说明在匿名内部类中成功调用到了外部类的方法，且这个调用不需要我们指定对象，jvm会帮我们自动完成。那么现在我们来看，这个外部类对象在内部类中是以什么样的形式存在的呢？

既然内部类持有了外部类的一个引用，那么我们只需要通过反射，将它的所有成员变量都打印出来就可以了。因此将test()方法改造如下：
```java
 private void test() {
    Callback callback = new Callback() {
        @Override
        public void onCall() {
            System.out.println(getName());
        }
    };
    callback.onCall();
    Class<? extends Callback> clazz = callback.getClass();
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        System.out.println(field.getName());
        try {
            System.out.println(field.get(callback));

        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```
运行代码，再次观察打印结果：
```java
小王
this$0
Test@4554617c
```
可以看到，在匿名内部类的实例callback中，外部类的引用是保存在this$0的指针中的。如果想要获得该对象的话，只需要反射该属性就可以了，接下来试试。
```java
private void test() {
    Callback callback = new Callback() {

        @Override
        public void onCall() {
            System.out.println(getName());
        }
    };
    callback.onCall();
    Class<? extends Callback> clazz = callback.getClass();

    Field field = null;
    try {
        field = clazz.getDeclaredField("this$0");
        System.out.println(field.getName());
            System.out.println(field.get(callback));
            Field this$0 = clazz.getDeclaredField(field.getName());
            Test test = (Test) this$0.get(callback);
            System.out.println(test.getName());

    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
}
```
运行代码，再次观察打印结果：
```java
小王
this$0
Test@4554617c
小王
```
可以看到，我们拿到了外部类的对象，并主动调用getName()方法打印出了结果。

这时候再思考一个问题，如果匿名内部类中也有一个对象叫做this\$0怎么办呢？这时候就不能通过直接`getDeclaredField("this$0")`来获取到外部类对象了。

熟悉反射的应该都知道，Class或者Field都有一个getModifiers();方法。modifier是Class或者Field的修饰符，记录了Class和Field的一些特性。另外还有一个Modifier类，提供了一些静态方法帮助判断Class和Field的一些特性。
```java
private void test() {
    Callback callback = new Callback() {
        private int this$0 = 0;
        @Override
        public void onCall() {
        }
    };
    callback.onCall();
    Class<? extends Callback> clazz = callback.getClass();
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        field.setAccessible(true);
        try {
            System.out.println("name: " + field.getName() + ",modifier:" + Long.toHexString(field.getModifiers())+",val:"+field.get(callback));
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```
打印结果如下：
```java
name: this$0,modifier:2,val:0
name: this$0$,modifier:1010,val:Test@4554617c
```
从中可以看到，外部类对象的名字变成了this$0$，而modifier的十六进制值为1010，查看Modifier中的静态变量，发现其实际上是
```java
public static final int FINAL = 0x00000010;
static final int SYNTHETIC = 0x00001000;
```
两者的或，其实就是表示Field或者Class是编译器自动生成的意思。这样一来，我们就可以通过modifier是否为0x1010来判断获取到的Field对象是否外外部类的引用了。
```java
public static final int FINAL = 0x00000010;
public static final int SYNTHETIC = 0x00001000;
public static final int SYNTHETIC_FINAL = FINAL | SYNTHETIC;

private void test() {
    Callback callback = new Callback() {
        private int this$0 = 0;

        @Override
        public void onCall() {

        }
    };
    callback.onCall();
    Class<? extends Callback> clazz = callback.getClass();
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        field.setAccessible(true);
        try {
            System.out.println("name: " + field.getName() + ",modifier:" + Long.toHexString(field.getModifiers()) + ",val:" + field.get(callback));
            if (checkModifier(field.getModifiers())){
                Field this$0 = clazz.getDeclaredField(field.getName());
                Test test = (Test) this$0.get(callback);
                System.out.println("调用了外部类方法:getName()="+ test.getName());
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }
}

private boolean checkModifier(int modifier) {
    if ((modifier & SYNTHETIC_FINAL) == SYNTHETIC_FINAL) {
        return true;
    }
    return false;
}
```
打印结果如下：
```java
name: this$0,modifier:2,val:0
name: this$0$,modifier:1010,val:Test@4554617c
调用了外部类方法:getName()=小王
```
成功获取到了外部类对象，并调用`getName()`方法！
