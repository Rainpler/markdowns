我们为什么要学习Java I/O?在对象序列化、Json解析、XML解析、zip压缩处理的时候，均需要以I/O作为基础，这些都需要很扎实的Java基础。

## Java I/O 概要设计
我们在I/O中经常能看见这样的使用方式：
```java
DataOutputStream out =new DataOutputStream(
              new BufferedOutputStream(
              new FileOutputStream(
              new File(file))));
```
一看是不是被绕晕了，这种嵌套的原理是什么？

我们先通过new File(file)打开了一个文件，然后通过FileOutputStream把它转化为文件输出流，接下来通过BufferedOutputStream创建一个字节缓冲输出流，当写文件的时候，能起到缓冲的作用，减少对磁盘的访问。最后写入数据的话,则必须指定好数据的输出格式，因此通过DataOutputStream，将Java基本数据类型写到底层输出流。

这个过程就像是人穿衣服一样，一层一层的穿上去，我们把I/O的这种设计模式称为装饰设计模式。
### 装饰设计模式
在装饰设计模式中，通常有那么两类接口，一类是Component对象的抽象父类，一类是Decorator装饰器的抽象父类。通过ConcreteComponent继承自Component实现具体的抽象对象,其内部持有一个Component对象。通过ConreteDecorator继承自Decorator，实现具体的装饰器对象和相应功能。

- Component：抽象构建接口
- ConcreteComponent：具体的构建对象，实现组件对象接口，通常就是被装饰的原始对象。就对这个对象添加功能。
- Decorator：所有装饰器的抽象父类，需要定义一个与组件接口一致的接口，内部持有一个Component对象，就是持有一个被装饰的对象。
- ConreteDecoratorA/ConreteDecoratorB：实际的装饰器对象，实现具体添加功能。
#### I/O中的装饰器模式
在I/O中，InputStream就是一个组件对象接口，其内部提供了一个``read()``方法，需要具体的构建对象去实现。
```java
public abstract class InputStream implements Closeable {

    public abstract int read() throws IOException;
}
```
而ByteArrayInputStream、ObjectInputStream、FileInputStream都是InputStream的具体构建对象。

```java
class FilterInputStream extends InputStream {

    protected volatile InputStream in;

    protected FilterInputStream(InputStream in) {
        this.in = in;
    }

    public int read() throws IOException {
        return in.read();
    }
```
FilterInputStream就是装饰器的抽象父类，内部持有了一个InputStream的对象，即为被修饰的对象。DataInputStream、BufferedInputStream都是其实际的装饰器对象。

### I/O学习的关键方法
>Java IO的学习是一件非常艰巨的任务。它的挑战是来自于要覆盖所有的可能性。不仅存在各种I/O源端还有想要和他通信的接收端（文件/控制台/网络链接），而且还需要以不同的方式与他们进行通信（顺序/随机存取/缓冲/二进制/字符/行/字 等等）这些情况综合起来就给我们带来了大量的学习任务，大量的类需要学习。
  
