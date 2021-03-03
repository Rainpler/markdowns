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

我们先通过new File(file)打开了一个文件，然后通过FileOutputStream把它转化为文件输出流，接下来通过BufferedOutputStream创建一个字节缓冲输出流，当写文件的时候，能起到缓冲的作用，减少对磁盘的访问。最后写入数据的话，则必须指定好数据的输出格式，因此通过DataOutputStream，将Java基本数据类型写到底层输出流。

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
}
```
FilterInputStream就是装饰器的抽象父类，内部持有了一个InputStream的对象，即为被修饰的对象。DataInputStream、BufferedInputStream等都是其实际的装饰器对象。

### I/O结构
Java IO的学习是一件非常艰巨的任务。它的挑战是来自于要覆盖所有的可能性。

不仅存在各种I/O源端还有想要和他通信的接收端（文件/控制台/网络链接），而且还需要以不同的方式与他们进行通信（顺序/随机存取/缓冲/二进制/字符/行/字 等等）这些情况综合起来就给我们带来了大量的学习任务，大量的类需要学习。

在I/O体系中，这些类总体可分为流式部分跟非流式部分。而流式部分又可细分为字节流和字符流。

#### 字节流
字节流是指传输过程中，传输数据的最基本单位是字节的流，一个不包含边界数据的连续流，一个连续的字节队列；字节流是由字节组成的，主要用在处理二进制数据。

在Java中，字节流是最基本的，所有的类都是派生自InputStream和OutputStream两个基类。

![](https://vkceyugu.cdn.bspapp.com/VKCEYUGU-b1ebbd3c-ca49-405b-957b-effe60782276/d3d26f0d-7ba4-432c-9a8d-8417ad907c0f.jpg)

#### 字符流
与字节流相对，传输数据的最基本单位是字符的流。

字节数据是二进制形式的，要转成我们能识别的正常字符，需要选择正确的编码方式，我们生活中遇到的乱码问题就是字节数据没有选择正确的编码方式来显示成字符。因此，跟字节流相比，字符流就是用于按照指定的编码格式去写（读）数据。

在传输方面上，由于计算机的传输本质都是字节，而一个字符由多个字节组成，转成字节之前先要去查表转成字节，所以传输时有时候会使用缓冲区。

在Java中，字符流的类结构如图，所有的类都是派生自Reader和Writer两个基类。

![字符流类结构](https://vkceyugu.cdn.bspapp.com/VKCEYUGU-b1ebbd3c-ca49-405b-957b-effe60782276/5b3516a1-d78f-4447-b47c-e6441044a24f.png)

Reader它会读取字节并通过特定的字符集编码成字符，Writer则相反。字符编码方式不同，有时候一个字符使用的字节数也不一样，比如ASCLL方式编码的字符，占一个字节；而UTF-8方式编码的字符，一个英文字符需要一个字节，一个中文需要三个字节。

我们观察InputStreamReader的源码可知，它首先先对InputStream先进行了一层封装，然后通过StreamDecoder，使用特定的charsetName，将字节流转化为字符流。
```java
public class InputStreamReader extends Reader {

    public InputStreamReader(InputStream in, String charsetName)
        throws UnsupportedEncodingException
    {
        super(in);
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        sd = StreamDecoder.forInputStreamReader(in, this, charsetName);
    }
}
```

它们的转换关系图如下：

![](https://vkceyugu.cdn.bspapp.com/VKCEYUGU-b1ebbd3c-ca49-405b-957b-effe60782276/1c682fe1-051a-4ea9-8ead-7e46cc8c4d85.jpg)

#### RandomAccessFile
File是平时很常见的读取文件的类，但是File只能从文件头开始读起。而RandomAccessFile不同，可以从文件的指定位置进行读写。

- 构造方法：RandomAccessFile raf = newRandomAccessFile(File file, String mode);
其中参数 mode 的值可选 "r"：可读，"w" ：可写，"rw"：可读性；

它提供了两个基本的成员方法。
- seek(int index);可以将指针移动到某个位置开始读写;
- setLength(long len);给写入文件预留空间；

#### NIO——FileChannel
Channel是对I/O操作的封装。

FileChannel配合着ByteBuffer，将读写的数据缓存到内存中，然后以批量/缓存的方式read/write，省去了非批量操作时的重复中间操作，操纵大文件时可以显著提高效率（和Stream以byte数组方式有什么区别？经过测试，效率上几乎无区别）。

基本使用
```java
private static void copyFileByFileChannel(File sourceFile, File targetFile){
		Instant begin = Instant.now();

		RandomAccessFile randomAccessSourceFile;
		RandomAccessFile randomAccessTargetFile;

		try {
			randomAccessSourceFile = new RandomAccessFile(sourceFile, "r");
			randomAccessTargetFile = new RandomAccessFile(targetFile, "rw");

		} catch (Exception e) {
			e.printStackTrace();
			return;
		}

		FileChannel sourceFileChannel = randomAccessSourceFile.getChannel();
		FileChannel targetFileChannel = randomAccessTargetFile.getChannel();

		ByteBuffer byteBuffer = ByteBuffer.allocate(1024*1024);
		try {
			while(sourceFileChannel.read(byteBuffer) != -1) {
				byteBuffer.flip();
				targetFileChannel.write(byteBuffer);
				byteBuffer.clear();
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			try {
				sourceFileChannel.close();
			} catch (Exception e2) {
				e2.printStackTrace();
			}

			try {
				targetFileChannel.close();
			}catch (Exception e) {
				e.printStackTrace();
			}
		}
		System.out.println("total spent: " + Duration.between(begin, Instant.now()).toMillis());
	}
```
