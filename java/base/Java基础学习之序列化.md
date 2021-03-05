## 序列化
>将数据结构或对象转换成二进制串的过程。
### 序列化方案
#### Serializeble Java序列化方案
在Java中使用Serializeble有两种方法，一种是实现Serializeble接口，另一种是实现Externalizable接口，它继承自Java.io.Serializeble类。
我们观察源码可以发现，Serializeble接口内部是没有实现的。
```java
public interface Serializeble {

}
```
实际上，Serializeble就相当于是一个flag标识，使用的时候需要通过ObjectOutputStream来进行序列化和ObjectInputStream进行反序列化。
来看这么一个使用的例子：
```java
//
public class demo01 {

    static class User implements Serializable {
        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String name;
        public int age;

        public String nickName;

        @Override
        public String toString() {
            return "User{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    ", nickName=" + nickName +
                    '}';
        }
    }

    /**
     * 序列化对象
     * @param obj
     * @param path
     * @return
     */
    synchronized public static boolean saveObject(Object obj, String path) {//持久化
        if (obj == null) {
            return false;
        }
        ObjectOutputStream oos = null;
        try {
            oos = new ObjectOutputStream(new FileOutputStream(path));// 创建序列化流对象
            oos.writeObject(obj);
            oos.close();
            return true;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (oos != null) {
                try {
                    oos.close(); // 释放资源
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return false;
    }
    /**
     * 反序列化对象
     *
     * @param path
     * @param <T>
     * @return
     */
    @SuppressWarnings("unchecked ")
    synchronized public static <T> T readObject(String path) {
        ObjectInputStream ojs = null;
        try {
            ojs = new ObjectInputStream(new FileInputStream(path));// 创建反序列化对象
            return (T) ojs.readObject();// 还原对象
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        } finally {
            if(ojs!=null){
                try {
                    ojs.close();// 释放资源
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return null;
    }

     public static void main(String[] args) {
       SerialTest();
     }

    public static String tmp = "D:\\xuliehuaTest\\";

    private static void SerialTest(){
        User user = new User("zero",18);
        saveObject(user,tmp+"a.out");
        System.out.println("1: " + user);
        user = readObject(tmp+"a.out");
        System.out.println("反序列化： 2: " + user);
    }
}

```
而Externalizable类内部包含了``writeExternal(ObjectOutput)``和``readExternal(ObjectInput)``两个方法。
```java
public interface Externalizable extends Java.io.Serializeble{

  void writeExternal(ObjectOutput out) throws IOExcption;

  void readExternal(ObjectInput in) throws IOExcption,ClassNotFoundException;
}
```
但是使用的时候要注意：
1. 读写的顺序要求一致
2. 读写的成员变量个数要求一致，否则会报EOFException异常
3. 必须要有一个无参构造函数

那么要怎么使用呢？
```java
/**
 * Externalizable 接口的作用
 */
public class demo02 {

    static class User implements Externalizable {
        //必须要一个public的无参构造函数
        public User(){}

        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String name;
        public int age;


        @Override
        public String toString() {
            return "User{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }

        // 1. 读写的顺序要求一致
        // 2. 读写的成员变量个数要求一致，否则会报EOFException异常
        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeObject(name);
            out.writeInt(age);
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            name = (String)in.readObject();
            age = in.readInt();
        }
    }

    public static String basePath = System.getProperty("user.dir") + "\\";
    public static String tmp = "D:\\xuliehuaTest\\";

    public static void main(String[] args) {
        ExternalableTest();
    }

    private static void ExternalableTest() {
        User user = new User("zero", 18);
        System.out.println("1: " + user);
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        ObjectOutputStream oos = null;
        byte[] userData = null;
        try {
            oos = new ObjectOutputStream(out);
            oos.writeObject(user);
            userData = out.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }


        ObjectInputStream ois = null;
        try {
            ois = new ObjectInputStream(new ByteArrayInputStream(userData));
            user = (User)ois.readObject();
            System.out.println("反序列化后 2: " + user);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```
##### 关于序列化的几个问题
1. 什么是serialVersionUID ？如果你不定义这个, 会发生什么？
2. 假设你有一个类,它序列化并存储在持久性中, 然后修改了该类以添加新字段。如果对已序列化的对象进行反序列化, 会发生什么情况？

serialVersionUID 是一个 private static final long 型 ID, 当它被印在对象上时，它通常是对象的哈希码,你可以使用 serialver 这个JDK工具来查看序列化对象的 serialVersionUID。
```java
static class User1 implements Serializable{

      private static final long serialVersionUID = 1;

      ...

}
```

SerialVerionUID 用于对象的版本控制。也可以在类文件中指定 serialVersionUID。不指定serialVersionUID的后果是，当你添加或修改类中的任何字段时，则已序列化类将无法恢复，因为为新类和旧序列化对象生成的 serialVersionUID 将有所不同，这时候将会报InvalidClassException的错误。Java 序列化过程依赖于正确的序列化对象恢复状态的，并在序列化对象序列版本不匹配的情况下引发。

3. 序列化时,你希望某些成员不要序列化？你如何实现它？

有时候也会变着形式问，比如问什么是瞬态 trasient 变量, 瞬态和静态变量会不会得到序列化等,所以,如果你不希望任何字段是对象的状态的一部分, 然后声明它静态或瞬态根据你的需要, 这样就不会是在 Java 序列化过程中被包含在内。
```java
 public transient String nickName;
```
比如User类中声明nickName为``transient``,则不会序列化该成员变量。

4. 如果类中的一个成员未实现可序列化接口, 会发生什么情况？

如果尝试序列化实现可序列化的类的对象,但该对象包含对不可序列化类的引用，则在运行时将引发不可序列化异常 NotSerializableException。

5. 如果类是可序列化的, 但其超类不是, 则反序列化后从超类继承的实例变量的状态如何？

Java 序列化过程仅在对象层级都是可序列化的类中继续， 即：实现了可序列化接口。 如果超类没有实现可序列化接口，则从超类继承的实例变量的值将通过调用构造函数初始化。

一旦构造函数链启动, 就不可能停止， 因此， 即使层次结构中更高的类成员变量实现了可序列化接口，也将通过执行构造函数创建，而不再是反序列化得到。

6. 是否可以自定义序列化过程, 或者是否可以覆盖 Java 中的默认序列化过程？

答案是肯定的, 你可以。我们都知道,对于序列化一个对象需调用`` ObjectOutputStream.writeObject(saveThisObject)``, 并用 ``ObjectInputStream.readObject()`` 读取对象, 但 Java 虚拟机为你提供的还有一件事, 是定义这两个方法。

如果在类中定义这两种方法, 则 JVM 将调用这两种方法, 而不是应用默认序列化机制。你可以在此处通过执行任何类型的预处理或后处理任务来自定义对象序列化和反序列化的行为。

需要注意的重要一点是要声明这些方法为私有方法, 以避免被继承、重写或重载。由于只有 Java 虚拟机可以调用类的私有方法, 你的类的完整性会得到保留, 并且 Java 序列化将正常工作。

7. 假设新类的超级类实现可序列化接口, 如何避免新类被序列化？

对于序列化一个对象需调用 ``ObjectOutputStream.writeObject(saveThisObject)``, 并用``ObjectInputStream.readObject()`` 读取对象, 但 Java 虚拟机为你提供的还有一件事, 是定义这两个方法。如果在类中定义这两种方法, 则 JVM 将调用这两种方法, 而不是应用默认序列化机制。你可以在此处通过执行任何类型的预处理或后处理任务来自定义对象序列化和反序列化的行为。

8. 在 Java 中的序列化和反序列化过程中使用哪些方法？

考察你是否熟悉`` readObject()`` 、``writeObject()``、``readExternal() ``和 ``writeExternal()``的用法。Java 序列化由java.io.ObjectOutputStream类完成。该类是一个筛选器流, 它封装在较低级别的字节流中, 以处理序列化机制。要通过序列化机制存储任何对象, 我们调用``ObjectOutputStream.writeObject(savethisobject)``, 若反序列化该对象, 我们使用``ObjectInputStream.readObject()``方法。

调用`` writeObject() ``方法在 java 中触发序列化过程。关于 ``readObject() ``方法, 需要注意的一点很重要一点是, 它用于从持久性读取字节, 并从这些字节创建对象, 并返回一个对象, 该对象需要类型强制转换为正确的类型。

Java的序列化机制针对枚举类型是特殊处理的。简单来讲，在序列化枚举类型时，只会存储枚举类的引用和枚举常量的名称。随后的反序列化的过程中，这些信息被用来在运行时环境中查找存在的枚举类型对象。
#### Parcelable Android独有的序列化方案
Parcelable是Android为我们提供的序列化的接口，跟Serializeble相比，Parcelable相使用相对复杂一些，但效率相对Serializable也高很多。
| Serializable               | Parcelable                       |
| -------------------------- | -------------------------------- |
| 通过IO对硬盘操作，速度较慢 | 直接在内存操作，效率高，性能好   |
| 大小不受限制               | 一般不能超过1M，修改内核也只能4M |
| 大量使用反射，产生内存碎片 |                                  |


先来看Parcelable的定义:
```java
public interface Parcelable
{
    //内容描述接口，基本不用管
    public int describeContents();
    //写入接口函数，打包
    public void writeToParcel(Parcel dest, int flags);
    //读取接口，目的是要从Parcel中构造一个实现了Parcelable的类的实例处理。因为实现类在这里还是不可知的，所以需要用到模板的方式，继承类名通过模板参数传入

    //为了能够实现模板参数的传入，这里定义Creator嵌入接口,内含两个接口函数分别返回单个和多个继承类实例
    public interface Creator<T>
    {
           public T createFromParcel(Parcel source);
           public T[] newArray(int size);
    }
}
```
我们需要实现Parcelable接口，并实现``describeContents()``和``writeToParcel(Parcel dest, int flags)``两个方法。``writeToParcel()``是读取接口，目的是要从Parcel中构造一个实现了Parcelable的类的实例处理。因为实现类在这里还是不可知的，所以需要用到模板的方式，继承类名通过模板参数传入。

所以，实现Parcelable步骤如下

1. implements Parcelable

2. 重写writeToParcel方法，将你的对象序列化为一个Parcel对象，即：将类的数据写入外部提供的Parcel中，打包需要传递的数据到Parcel容器保存，以便从 Parcel容器获取数据

3. 重写describeContents方法，内容接口描述，默认返回0就可以

4. 实例化静态内部对象CREATOR实现接口Parcelable.Creator

public static final Parcelable.Creator\<T> CREATOR
注：其中public static final一个都不能少，内部对象CREATOR的名称也不能改变，必须全部大写。需重写本接口中的两个方法：createFromParcel(Parcel in) 实现从Parcel容器中读取传递数据值，封装成Parcelable对象返回逻辑层，newArray(int size) 创建一个类型为T，长度为size的数组，仅一句话即可（return new T[size]），供外部类反序列化本类数组使用。
```java
public class Course implements Parcelable {

    private String name;
    private float score;

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
        dest.writeFloat(this.score);
    }

    protected Course(Parcel in) {
        this.name = in.readString();
        this.score = in.readFloat();
    }

    public static final Parcelable.Creator<Course> CREATOR = new Parcelable.Creator<Course>() {
        @Override
        public Course createFromParcel(Parcel source) {
            return new Course(source);
        }

        @Override
        public Course[] newArray(int size) {
            return new Course[size];
        }
    };
}

```
简而言之：通过writeToParcel将你的对象映射成Parcel对象，再通过createFromParcel将Parcel对象映射成你的对象。也可以将Parcel看成是一个流，通过writeToParcel把对象写到流里面，在通过createFromParcel从流里读取对象，只不过这个过程需要你来实现，因此写的顺序和读的顺序必须一致。


### 面试相关
1. 反序列化后的对象，需要调用构造函数重新构造吗

答：不会, 因为是从二进制直接解析出来的. 适用的是 Object 进行接收再强转, 因此不是原来的那个对象

2. 序列前的对象与序列化后的对象是什么关系？是("=="还是equal？是浅复制还是深复制？)

答：是一个深拷贝, 前后对象的引用地址不同，但是有一个例外是枚举，枚举类在被序列化的时候，存储起来的对象是枚举的引用和名称，反序列化的时候，从运行环境中查找枚举对象。

3. Android里面为什么要设计出Bundle而不是直接用Map结构

答:bundle 内部适用的是 ArrayMap, ArrayMap 相比 Hashmap 的优点是, 扩容方便, 每次扩容是原容量的一半, 在［百量］ 级别, 通过二分法查找 key 和 value (ArrayMap 有两个数组, 一个存放 key 的 hashcode, 一个存放 key+value 的 Entry) 的效率要比 hashmap 快很多, 由于在内存中或者 Android 内部传输中一般数据量较小, 因此用 bundle 更为合适。

4. SerialVersionID的作用是什么？

答:用于数据的版本控制, 如果反序列化后发现 ID 不一样, 认为不是之前序列化的对象。

5. Android 中 intent/bundle 的通信原理以及大小限制?

答: Android 中的 bundle 实现了 parcelable 的序列化接口, 目的是为了在进程间进行通讯, 不同的进程共享一片固定大 小的内存, parcelable 利用 parcel 对象的 read/write 方法, 对需要传递的数据进行内存读写, 因此这一块共享内存不能 过大, 在利用 bundle 进行传输时, 会初始化一个 BINDER_VM_SIZE 的大小 = 1 * 1024 * 1024 - 4096 * 2, 即便通过 修改 Framework 的代码, bundle 内核的映射只有 4M, 最大只能扩展到 4M.

6. 为何Intent不能直接在组件间传递对象而要通过序列化机制？

答: 因为 Activity 启动过程是需要与 AMS 交互, AMS 与 UI 进程是不同一个的, 因此进程间需要交互数据, 就必须序列化。

7. 序列化与持久化的关系和区别是什么？

答: 序列化是为了进程间数据交互而设计的, 持久化是为了把数据存储下来而设计的。
