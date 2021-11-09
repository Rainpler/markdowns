> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/828d760ec54b)

#### 目录

![](http://upload-images.jianshu.io/upload_images/8850933-cb8294aa0cca7937.png)

#### 前期准备

###### 1. 工具下载

这里我把需要用到的代码和工具都整理了一下放到了一起：[https://www.aliyundrive.com/s/ALCxbGeWY2o](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.aliyundrive.com%2Fs%2FALCxbGeWY2o)

![](http://upload-images.jianshu.io/upload_images/8850933-809ae0eb076eccdf.png)

**bzip2：**

是 bsdiff 依赖的一个库，这里我只存放了需要用到的文件，完整版的下载地址为：

[https://sourceforge.net/projects/bzip2/files/latest/download](https://links.jianshu.com/go?to=https%3A%2F%2Fsourceforge.net%2Fprojects%2Fbzip2%2Ffiles%2Flatest%2Fdownload)

**bsdiff-win：**

是编译好的 Windows 平台下的可执行文件，可以在 Windows 平台生成差异文件和合并文件

**bsdiff-source：**

是 bsdiff 的源码，它的官网为：

[http://www.daemonology.net/bsdiff/](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.daemonology.net%2Fbsdiff%2F)

###### 2. 工具的使用方法

只需要在工具所在的目录打开命令行窗口

![](http://upload-images.jianshu.io/upload_images/8850933-5d11a801180ebdee.png)

然后输入命令即可

```
#生成差异文件命令
bsdiff [旧文件] [新文件] [差异文件]
#合并文件命令
bspatch [旧文件] [新文件] [差异文件]
```

例如我这创建两个文本文件 old.txt 和 new.txt

![](http://upload-images.jianshu.io/upload_images/8850933-40a6aac2b8b957e5.png) old.txt

![](http://upload-images.jianshu.io/upload_images/8850933-eb164bf6f94753d9.png) new.txt

然后我可以利用 bsdiff 命令生成差异文件

![](http://upload-images.jianshu.io/upload_images/8850933-86a3cea54a3a4463.png)

![](http://upload-images.jianshu.io/upload_images/8850933-9578af8f1dbd5dfe.png)

这个时候我再利用 bspatch 命令，将 old.txt 和 patch 文件合成 new2.txt

![](http://upload-images.jianshu.io/upload_images/8850933-11c9e1953fc2856c.png)

![](http://upload-images.jianshu.io/upload_images/8850933-43681692ba26b2f5.png)

我们打开 new2.txt 发现与 new.txt 是一样的

![](http://upload-images.jianshu.io/upload_images/8850933-0558588315f90b74.png) new2.txt

#### 原理讲解

实现原理其实就是将新 APK 文件与旧 APK 文件进行对比，得出一个差异文件，然后用户端下载这个差异文件与手机上的那个旧 APK 文件进行合并即可得到与新的 APK 文件一样的文件，然后再安装这个新 APK 即可实现增量更新，如下图所示

![](http://upload-images.jianshu.io/upload_images/8850933-76469e23441c3abf.png)

#### 具体实现

###### 1. 集成 bspatch 到项目

由于 Android 端只需要合并文件所以我们只需要集成 bspatch 即可，我们将 bsdiff-source/bsdiff-4.3 文件夹中的 bspatch.c 文件拷贝到 cpp 目录，然后将 bzip2 文件夹下的文件拷贝到 cpp 下的 bzip（新建的目录）目录下

![](http://upload-images.jianshu.io/upload_images/8850933-29727fad4bc92003.png)

此外我们还要对 bspatch.c 进行修改，我们在文件顶部加入 bzip2 的引用

```c
/** 导入bzip2的引用*/
#include "bzip/bzlib.c"
#include "bzip/crctable.c"
#include "bzip/compress.c"
#include "bzip/decompress.c"
#include "bzip/randtable.c"
#include "bzip/blocksort.c"
#include "bzip/huffman.c"
```

![](http://upload-images.jianshu.io/upload_images/8850933-ef6232ccdbc89c17.png)

否则的话你运行项目的时候可能会报如下错误

![](http://upload-images.jianshu.io/upload_images/8850933-48e255f12b22f848.png)

然后我们还需要新建 bspatch.h 放到 bzip 文件夹下，这样做目的是为了可以在 native-lib.cpp 文件中使用 main 方法（注意：这里的 main 方法并不是入口函数，就是一个执行命令的普通函数）

![](http://upload-images.jianshu.io/upload_images/8850933-512ab05cfcb05c7d.png)

bspatch.h 文件如下

```c
#ifndef INCREMENTUPDATEDEMO_BSPATCH_H
#define INCREMENTUPDATEDEMO_BSPATCH_H
int main(int argc,char * argv[]);
#endif //INCREMENTUPDATEDEMO_BSPATCH_H
```

然后在 bspatch.c 中引入 bspatch.h 头文件

![](http://upload-images.jianshu.io/upload_images/8850933-1191fb1a30712f83.png)

接下来我们需要配置下 CMakeLists.txt 文件将 bzip 下的 c 文件和. h 头文件链接到项目

```j
cmake_minimum_required(VERSION 3.10.2)

project("incrementupdatedemo")
#定义一个全局变量包含了所有要编译的C文件
file(GLOB BZIP bzip/*.c)
#导入头文件
include_directories(bzip)
add_library( # Sets the name of the library.
             native-lib
             SHARED
             native-lib.cpp
             #将bzip下的.c文件添加到library
             BZIP)
find_library( # Sets the name of the path variable.
              log-lib
              log )
target_link_libraries( # Specifies the target library.
                       native-lib
                       ${log-lib} )
```

###### 2. 创建 JNI 方法

创建 PatchUtil 工具类，创建合并文件的 JNI 方法

```java
public class PatchUtil {
    static {
        System.loadLibrary("native-lib");
    }
    /**
     * 合并APK文件
     * @param oldApkFile 旧APK文件路径
     * @param newApkFile 新APK文件路径（存储生成的APK的路径）
     * @param patchFile 差异文件
     */
    public native static void patchAPK(String oldApkFile,String newApkFile,String patchFile);
}
```

C++ 实现 JNI 方法

```c
#include <jni.h>
#include <string>
#include "bspatch.h"

//extern  C++调用C 存在兼容性问题
extern "C"
JNIEXPORT void JNICALL
Java_com_itfitness_incrementupdatedemo_PatchUtil_patchAPK(JNIEnv *env, jclass clazz,
                                                          jstring old_apk_file,
                                                          jstring new_apk_file,
                                                          jstring patch_file) {
    int argc = 4;
    char * argv[argc];
    argv[0] = "bspatch";
    argv[1] = (char*) (env->GetStringUTFChars(old_apk_file, 0));
    argv[2] = (char*) (env->GetStringUTFChars(new_apk_file, 0));
    argv[3] = (char*) (env->GetStringUTFChars(patch_file, 0));

    // 调用合并的方法
    main(argc, argv);

    // 用完之后要释放
    env->ReleaseStringUTFChars(old_apk_file, argv[1]);
    env->ReleaseStringUTFChars(new_apk_file, argv[2]);
    env->ReleaseStringUTFChars(patch_file, argv[3]);
}
```

###### 3.Activity 中增加合成的调用

```java
public class MainActivity extends AppCompatActivity {
    private TextView tv_version;
    private Button bt_update;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        tv_version = findViewById(R.id.tv_version);
        bt_update = findViewById(R.id.bt_update);
        tv_version.setText("1.0");
        bt_update.setOnClickListener(v->{
            new Thread(() -> {
                File oldApkFile = new File(getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), "old.apk");
                File newApkFile = new File(getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), "new.apk");
                File patchFile = new File(getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), "patch");
                PatchUtil.patchAPK(oldApkFile.getAbsolutePath(),newApkFile.getAbsolutePath(),patchFile.getAbsolutePath());
                //安装APK
                install(newApkFile);
            }).start();
        });
    }

    private void install(File file) {
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) { // 7.0+以上版本
            Uri apkUri = FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", file);
            intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            intent.setDataAndType(apkUri, "application/vnd.android.package-archive");
        } else {
            intent.setDataAndType(Uri.fromFile(file), "application/vnd.android.package-archive");
        }
        startActivity(intent);
    }
}
```

###### 4. 编译生成旧 APK

这里我们在 1.0 版本（旧版本）中展示当前应用的版本号，如下所示

![](http://upload-images.jianshu.io/upload_images/8850933-c0e9214a19fd92c8.png)

然后我们取出编译生成的 APK 命名为 old.apk

![](http://upload-images.jianshu.io/upload_images/8850933-a37ff5a7d7d4f698.png)

###### 5. 编译生成新 APK

然后我们将版本号改为 2.0 并且在按钮下增加一张图片

![](http://upload-images.jianshu.io/upload_images/8850933-cb2035afd77d8672.png)

我们将其命名为 new.apk

![](http://upload-images.jianshu.io/upload_images/8850933-9dd506d2ef57e129.png)

###### 6. 使用 bsdiff 生成差异文件

使用 bsdiff 生成差异文件

![](http://upload-images.jianshu.io/upload_images/8850933-ab6fbf3c2c84d770.png)

![](http://upload-images.jianshu.io/upload_images/8850933-307ae4e7122b4f49.png)

###### 7. 使用 bspatch 合并文件

接下来我们装回旧版然后将 old.apk 和 patch 放到 SD 卡中，当然真实环境的 patch 文件是通过网络下载得到的，这里我们模拟已经下载完毕了

![](http://upload-images.jianshu.io/upload_images/8850933-8aab16c5a0620c0c.png)

点击按钮，会发现 Download 文件夹下出现了一个 new.apk 文件

![](http://upload-images.jianshu.io/upload_images/8850933-3871b5e8674d6f4b.png)

然后我们通过代码将 new.apk 安装到手机上

![](http://upload-images.jianshu.io/upload_images/8850933-2de8c6506d5ca151.gif)

#### 案例源码

[https://gitee.com/itfitness/increment-update](https://links.jianshu.com/go?to=https%3A%2F%2Fgitee.com%2Fitfitness%2Fincrement-update)
