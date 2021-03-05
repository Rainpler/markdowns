### APK文件反编译
首先，我们先来了解一下什么是编译。我们编写的Java代码，称之为高级语言，因为这是程序员认识的语言。但是我们的计算机，是不认识这样的语言的，因此，高级语言就需要通过编译，转成低级语言让计算机去执行。

相反的，反编译就是，从低级语言反向工程，去获取其源代码的过程。

我们虽然很难将机器语言反编译成源代码，但是，我们还是可以把中间代码进行反编译的。就像我们虽然不能把经过虚拟机编译后的机器语言进行反编译，但是我们把javac编译得到的class进行反编译还是可行的。

所以，我们说Java的反编译，一般是将class文件转换成java文件。
#### 怎么进行反编译
我们来看了解apk的文件结构，通过修改apk文件后缀名为zip，解压得到文件夹，可以看到以下几个文件（文件夹）
![](https://ae01.alicdn.com/kf/U25dec2ae421d47eba64982fb1d1d5ff0o.jpg)
- META-INF 签名文件
- res 资源文件夹
- AndroidManifest.xml 清单文件
- classes.dex  Dalvik可执行文件，里面包含了app中的所有源码，因此反编译就是通过该文件获取java源码
- resources.arsc app的资源索引表

使用dex2jar可以将classes.dex转变为jar包，再通过jd-gui就可以查看class文件的内容了。但是现在的Andriod开发，为了防止反编译，都会进行代码混淆和加固，以防止代码的泄漏。
### APK加固的方案原理

Apk加固有多种方案，这里就简单提到几种常见的。
#### 加固方案的手段
1. 反模拟器

模拟器运行apk，可以用模拟器监控到apk的各种行为，所以在实际的加固apk运行中，一旦发现模拟器在运行该APK，就停止核心代码的运行。

2. 代码虚拟化

代码虚拟化在桌面平台应用保护中已经是非常的常见了，主要的思路是自建一个虚拟执行引擎，然后把原生的可执行代码转换成自定义的指令进行虚拟执行。

3. 加密

样本的部分可执行代码是以压缩或者加密的形式存在的，比如，被保护过的代码被切割成多个小段，前面的一段代码先把后面的代码片段在内存中解密，然后再去执行解密之后的代码，如此一块块的迭代执行。

### AES加密项目实战
这里，我们利用dex加壳的例子来进行加固实战。

#### 加固总体框架
对于一个apk，我们对它进行解压，得到dex源文件和其他文件。对于dex源文件进行AES加密，我们要把文件里面的二进制数据都进行AES加密，然后再写回到dex文件中。这里面涉及到大量的I/O。

我们再有一个dex壳文件，然后与dex源文件合并，重新打包成apk，并重新签名。

那么，在这里我们还有几个问题？
- dex文件可以随便拼凑吗？
- 壳dex 怎么来的
- 如何签名？
- 如何运行新apk（如何脱壳）？

加固的目的是保护dex，直接而言就是对dex文件进行操作，对dex文件动刀子，就像外科医生动手术一样，我们必须掌握dex的构造。

**Dex文件结构**

| header | string_ids   | type_ids   | proto_ids   | field_ids | metod_ids  | class_defs | data   | link_data  |
| ------ | ------------ | ---------- | -------------- | --------- | ------ | ---------- | ------ | ---------- |
| 文件头 | 字符串的索引 | 类型的索引 | 方法原型的索引 | 域的索引  | 方法的索引 | 类的定义区 | 数据区 | 链接数据区 |


在一个dex文件的文件头里面，包含了一系列的文件信息，包括文件签名，文件大小等等。
```c++
strut header_item{
   ubyte[8]   magic;
   unit       checksum;
   ubyte[20]  signature;
   unit       file_size;
   unit       header_size;
   unit       endian_tag;
   ....
}
```
我们了解了dex文件的构造，就可以在相应的文件区域进行修改，而不破坏dex文件的整体结构。

**Android 的打包流程**
在我们平时的开发中，使用的都是Android Studio可视化工具，可以很方便的进行打包，因为AS将这些工作都封装起来了，我们只需要直接build就可以了。

实际上，在我们的sdk目录下，有一个build-tool文件夹，里面包含了一系列的打包工具，包括**aapt、aidl、dx**等工具。当我们打包的时候，aapt就会将resource中中的资源文件打包成R.java文件，R.java会自动收录当前应用中所有的资源，并根据这些资源建立对应的ID。如果我们有编写aidl文件的话，aidl工具就会将其打包成Java的接口。再加上我们应用本身的源文件，经过javac编译，生成.class文件。

得到.class文件后，再通过dx.bat，就可以将其转换为dex文件了。前面我们的资源虽然经过打包，但得到的R.java文件其实类似于资源字典，真正的资源文件还在。资源文件加上dex文件，和其他文件，经过apkBuilder工具打包成apk包。但是这时候的apk包还要经过apksigner签名，才可以运行。


#### 加固实战
这里进行的加固只是一个粗粒度的加固，共分为三步进行：
#####  1. 处理原始apk 加密dex
```java
    AES.init(AES.DEFAULT_PWD);
    //解压apk
    File apkFile = new File("source/apk/app-debug.apk");
    File newApkFile = new File(apkFile.getParent() + File.separator + "temp");
    if(!newApkFile.exists()) {
    	newApkFile.mkdirs();
    }
//        Zip.unZip(apkFile,newApkFile);
    File mainDexFile = AES.encryptAPKFile(apkFile,newApkFile);
    if (newApkFile.isDirectory()) {
    	File[] listFiles = newApkFile.listFiles();
    	for (File file : listFiles) {
    		if (file.isFile()) {
    			if (file.getName().endsWith(".dex")) {
    				String name = file.getName();
    				System.out.println("rename step1:"+name);
    				int cursor = name.indexOf(".dex");
    				String newName = file.getParent()+ File.separator + name.substring(0, cursor) + "_" + ".dex";
    				System.out.println("rename step2:"+newName);
    				file.renameTo(new File(newName));
    			}
		    }
      }
```

#####  2. 处理aar 获得壳dex
aar为原Android工程的模块打包，包含了项目的application
```java
  File aarFile = new File("source/aar/mylibrary-debug.aar");
  File aarDex  = Dx.jar2Dex(aarFile);
  //        aarData = Utils.getBytes(aarDex);   //将dex文件读到byte 数组

  File tempMainDex = new File(newApkFile.getPath() + File.separator + "classes.dex");
  if (!tempMainDex.exists()) {
      tempMainDex.createNewFile();
  }
  //        System.out.println("MyMain" + tempMainDex.getAbsolutePath());
  FileOutputStream fos = new FileOutputStream(tempMainDex);
  byte[] fbytes = Utils.getBytes(aarDex);
  fos.write(fbytes);
  fos.flush();
  fos.close();
```
#####  3. 打包签名
```java
/**
     * 第3步 打包签名
     */
    File unsignedApk = new File("result/apk-unsigned.apk");
    unsignedApk.getParentFile().mkdirs();
//        File disFile = new File(apkFile.getAbsolutePath() + File.separator+ "temp");
    Zip.zip(newApkFile, unsignedApk);
    //不用插件就不能自动使用原apk的签名...
    File signedApk = new File("result/apk-signed.apk");
    Signature.signature(unsignedApk, signedApk);
```
在``AES.encryptAPKFile``方法中,进行了文件的解压和加密
```java
/**
    *
    * @param srcApkFile  源文件所在位置
    * @param dstApkFile  目标文件
    * @return             加密后的新dex 文件
    * @throws Exception
    */
    public static File encryptAPKFile(File srcApkFile, File dstApkFile) throws Exception {
        if (srcApkFile == null) {
        	System.out.println("encryptAPKFile :srcAPKfile null");
            return null;
        }
//        File disFile = new File(srcAPKfile.getAbsolutePath() + "unzip");
//		Zip.unZip(srcAPKfile, disFile);
        Zip.unZip(srcApkFile, dstApkFile);
        //获得所有的dex （需要处理分包情况）
        File[] dexFiles = dstApkFile.listFiles(new FilenameFilter() {
            @Override
            public boolean accept(File file, String s) {
                return s.endsWith(".dex");
            }
        });

        File mainDexFile = null;
        byte[] mainDexData = null;

        for (File dexFile: dexFiles) {
        	//读数据
            byte[] buffer = Utils.getBytes(dexFile);
            //加密
            byte[] encryptBytes = AES.encrypt(buffer);

            if (dexFile.getName().endsWith("classes.dex")) {
                mainDexData = encryptBytes;
                mainDexFile = dexFile;
            }
           //写数据  替换原来的数据
            FileOutputStream fos = new FileOutputStream(dexFile);
            fos.write(encryptBytes);
            fos.flush();
            fos.close();
        }
        return mainDexFile;
    }

    public static byte[] encrypt(byte[] content) {
        try {
            byte[] result = encryptCipher.doFinal(content);
            return result;
        } catch (IllegalBlockSizeException e) {
            e.printStackTrace();
        } catch (BadPaddingException e) {
            e.printStackTrace();
        }
        return null;
    }
```
经过这样处理，我们得到的apk中，就有两部分dex构成，一部门是经过加密的源dex文件，一部分是未经加密的dex。当我们运行应用的时候，未经加密的dex先被加载，再对加密的源dex文件进行解密，再加载它。

**ShellApplication.java**
```java

public class ShellApplication extends Application {
    private static final String TAG = "ShellApplication";

    public static String getPassword(){
        return "abcdefghijklmnop";
    }

//    static {
//        System.loadLibrary("native-lib");
//    }

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);


        AES.init(getPassword());
        File apkFile = new File(getApplicationInfo().sourceDir);
        //data/data/包名/files/fake_apk/
        File unZipFile = getDir("fake_apk", MODE_PRIVATE);
        File app = new File(unZipFile, "app");
        if (!app.exists()) {
            Zip.unZip(apkFile, app);
            File[] files = app.listFiles();
            for (File file : files) {
                String name = file.getName();
                if (name.equals("classes.dex")) {

                } else if (name.endsWith(".dex")) {
                    try {
                            byte[] bytes = getBytes(file);
                            FileOutputStream fos = new FileOutputStream(file);
                            byte[] decrypt = AES.decrypt(bytes);
//                        fos.write(bytes);
                            fos.write(decrypt);
                            fos.flush();
                            fos.close();
                        } catch (Exception e) {
                            e.printStackTrace();
                    }
                }
            }
        }
        List list = new ArrayList<>();
        Log.d("FAKE", Arrays.toString(app.listFiles()));
        for (File file : app.listFiles()) {
            if (file.getName().endsWith(".dex")) {
                list.add(file);
            }
        }

        Log.d("FAKE", list.toString());
        try {
            V19.install(getClassLoader(), list, unZipFile);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }

    private static Field findField(Object instance, String name) throws NoSuchFieldException {
        Class clazz = instance.getClass();

        while (clazz != null) {
            try {
                Field e = clazz.getDeclaredField(name);
                if (!e.isAccessible()) {
                    e.setAccessible(true);
                }

                return e;
            } catch (NoSuchFieldException var4) {
                clazz = clazz.getSuperclass();
            }
        }

        throw new NoSuchFieldException("Field " + name + " not found in " + instance.getClass());
    }

    private static Method findMethod(Object instance, String name, Class... parameterTypes)
            throws NoSuchMethodException {
        Class clazz = instance.getClass();
//        Method[] declaredMethods = clazz.getDeclaredMethods();
//        System.out.println("  findMethod ");
//        for (Method m : declaredMethods) {
//            System.out.print(m.getName() + "  : ");
//            Class<?>[] parameterTypes1 = m.getParameterTypes();
//            for (Class clazz1 : parameterTypes1) {
//                System.out.print(clazz1.getName() + " ");
//            }
//            System.out.println("");
//        }
        while (clazz != null) {
            try {
                Method e = clazz.getDeclaredMethod(name, parameterTypes);
                if (!e.isAccessible()) {
                    e.setAccessible(true);
                }

                return e;
            } catch (NoSuchMethodException var5) {
                clazz = clazz.getSuperclass();
            }
        }
        throw new NoSuchMethodException("Method " + name + " with parameters " + Arrays.asList
                (parameterTypes) + " not found in " + instance.getClass());
    }

    private static void expandFieldArray(Object instance, String fieldName, Object[]
            extraElements) throws NoSuchFieldException, IllegalArgumentException,
            IllegalAccessException {
        Field jlrField = findField(instance, fieldName);
        Object[] original = (Object[]) ((Object[]) jlrField.get(instance));
        Object[] combined = (Object[]) ((Object[]) Array.newInstance(original.getClass()
                .getComponentType(), original.length + extraElements.length));
        System.arraycopy(original, 0, combined, 0, original.length);
        System.arraycopy(extraElements, 0, combined, original.length, extraElements.length);
        jlrField.set(instance, combined);
    }

    private static final class V19 {
        private V19() {
        }

        private static void install(ClassLoader loader, List<File> additionalClassPathEntries,
                                    File optimizedDirectory) throws IllegalArgumentException,
                IllegalAccessException, NoSuchFieldException, InvocationTargetException,
                NoSuchMethodException {

            Field pathListField = findField(loader, "pathList");
            Object dexPathList = pathListField.get(loader);
            ArrayList suppressedExceptions = new ArrayList();
            Log.d(TAG, "Build.VERSION.SDK_INT " + Build.VERSION.SDK_INT);
            if (Build.VERSION.SDK_INT >= 23) {
                expandFieldArray(dexPathList, "dexElements", makePathElements(dexPathList, new
                                ArrayList(additionalClassPathEntries), optimizedDirectory,
                        suppressedExceptions));
            } else {
                expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new
                                ArrayList(additionalClassPathEntries), optimizedDirectory,
                        suppressedExceptions));
            }

            if (suppressedExceptions.size() > 0) {
                Iterator suppressedExceptionsField = suppressedExceptions.iterator();

                while (suppressedExceptionsField.hasNext()) {
                    IOException dexElementsSuppressedExceptions = (IOException)
                            suppressedExceptionsField.next();
                    Log.w("MultiDex", "Exception in makeDexElement",
                            dexElementsSuppressedExceptions);
                }

                Field suppressedExceptionsField1 = findField(loader,
                        "dexElementsSuppressedExceptions");
                IOException[] dexElementsSuppressedExceptions1 = (IOException[]) ((IOException[])
                        suppressedExceptionsField1.get(loader));
                if (dexElementsSuppressedExceptions1 == null) {
                    dexElementsSuppressedExceptions1 = (IOException[]) suppressedExceptions
                            .toArray(new IOException[suppressedExceptions.size()]);
                } else {
                    IOException[] combined = new IOException[suppressedExceptions.size() +
                            dexElementsSuppressedExceptions1.length];
                    suppressedExceptions.toArray(combined);
                    System.arraycopy(dexElementsSuppressedExceptions1, 0, combined,
                            suppressedExceptions.size(), dexElementsSuppressedExceptions1.length);
                    dexElementsSuppressedExceptions1 = combined;
                }

                suppressedExceptionsField1.set(loader, dexElementsSuppressedExceptions1);
            }

        }

        private static Object[] makeDexElements(Object dexPathList,
                                                ArrayList<File> files, File
                                                        optimizedDirectory,
                                                ArrayList<IOException> suppressedExceptions) throws
                IllegalAccessException, InvocationTargetException, NoSuchMethodException {

                Method makeDexElements = findMethod(dexPathList, "makeDexElements", new
                        Class[]{ArrayList.class, File.class, ArrayList.class});
                return ((Object[]) makeDexElements.invoke(dexPathList, new Object[]{files,
                        optimizedDirectory, suppressedExceptions}));
           }
    }

    /**
     * A wrapper around
     * {@code private static final dalvik.system.DexPathList#makePathElements}.
     */
    private static Object[] makePathElements(
            Object dexPathList, ArrayList<File> files, File optimizedDirectory,
            ArrayList<IOException> suppressedExceptions)
            throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {

        Method makePathElements;
        try {
            makePathElements = findMethod(dexPathList, "makePathElements", List.class, File.class,
                    List.class);
        } catch (NoSuchMethodException e) {
            Log.e(TAG, "NoSuchMethodException: makePathElements(List,File,List) failure");
            try {
                makePathElements = findMethod(dexPathList, "makePathElements", ArrayList.class, File.class, ArrayList.class);
            } catch (NoSuchMethodException e1) {
                Log.e(TAG, "NoSuchMethodException: makeDexElements(ArrayList,File,ArrayList) failure");
                try {
                    Log.e(TAG, "NoSuchMethodException: try use v19 instead");
                    return V19.makeDexElements(dexPathList, files, optimizedDirectory, suppressedExceptions);
                } catch (NoSuchMethodException e2) {
                    Log.e(TAG, "NoSuchMethodException: makeDexElements(List,File,List) failure");
                    throw e2;
                }
            }
        }
        return (Object[]) makePathElements.invoke(dexPathList, files, optimizedDirectory, suppressedExceptions);
    }

    private byte[] getBytes(File file) throws Exception {
        RandomAccessFile r = new RandomAccessFile(file, "r");
        byte[] buffer = new byte[(int) r.length()];
        r.readFully(buffer);
        r.close();
        return buffer;
    }
}

```
