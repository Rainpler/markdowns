# RxActivityResult
<strong>一种优雅的方式实现==startActivityForResult==，将Android中的startActivityForResult()事件转换为Rx事件流</strong>  

github地址：[https://github.com/VictorAlbertos/RxActivityResult](https://github.com/VictorAlbertos/RxActivityResult)  
作者：VictorAlbertos
## 常规写法
- MainResultActivity(Result的请求者)
- SecondResultActivity（Result的发送者）
```
sequenceDiagram
MainResultActivity->>SecondResultAcitivity: requestCode
SecondResultAcitivity->>MainResultActivity: result
```  

**MainResultActivity**
```java
public class MainResultActivity extends AppCompatActivity {

    //我们需要自己写一个常量作为requestCode，在请求result时传递进去
    public static final int REQUEST_CODE_NORMAL = 100;

    //我们省略其他无关紧要代码
    //打开新的界面，请求result
    public void startByNormal() {
        startActivityForResult(new Intent(this,
                        SecondResultActivity.class),
                REQUEST_CODE_NORMAL);
    }

    //获得Result数据并处理
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_CODE_NORMAL) {
            String content = data.getStringExtra("content");
            tvResult.setText("传回来的内容：");
            tvResult.append(content);
        }
    }

}
```
**SecondResultActivity**
```java
public class SecondResultActivity extends AppCompatActivity {   

    //我们省略其他无关紧要代码
    //发送Result数据给请求方，然后finish（）
     public void commitResult() {
        Intent intent = new Intent(this,MainResultActivity.class);
        intent.putExtra("content",etContent.getText().toString());
        setResult(1,intent);
        finish();
    }
}
```


## ==RxActivityResult==
- 添加依赖
   1. 在Project级别的build.gradle中添加：
   ```
   allprojects {
    repositories {
        jcenter()
        maven { url "https://jitpack.io" }
    }
   }
   ```
   2. 在module级别的build.gradle中添加：
   ```
    dependencies {
        compile 'com.github.VictorAlbertos:RxActivityResult:0.4.5-2.x'
        compile 'io.reactivex.rxjava2:rxjava:2.0.5'
    }
   ```
- MainResultActivity
```JAVA
public class MainResultActivity extends AppCompatActivity {
  //这是一个Button点击后调用的方法：
  //打开新的界面，请求result,并进行数据结果的处理
  public void startByRxActivityResult() {
        RxActivityResult.on(this)
                .startIntent(new Intent(this, SecondResultActivity.class))//请求result
                .map(result -> result.data())//对result的处理，转换为intent
                .subscribe(intent -> showResultIntentData(intent));//处理数据结果
  }

  //处理数据结果
  public void showResultIntentData(Intent data) {
      String content = data.getStringExtra("content");
      tvResult.setText("传回来的内容：");
      tvResult.append(content);
  }
}
```
**SecondResultActivity中的处理不变。
startByRxActivityResult()方法中，一行代码的链式调用即可完成：**

<blockquote>  

① 打开新的界面，请求result  
② 进行数据结果的处理  
③ 不需要自己实现一个常量作为requestCode，并在请求result时传递进去  
</blockquote>


**在subscribe()的onNext()回调中返回的Result对象是作者封装的一个类，我们可以从中取得很多东西：**
```JAVA
public class Result<T> {
    private final T targetUI;//订阅事件发生时所在的容器，本文中为MainResultActivity.
    private final int resultCode;//resultCode
    private final int requestCode;//requestCode
    private final Intent data;//存储数据的Intent对象

    public Result(T targetUI, int requestCode, int resultCode, Intent data) {
        this.targetUI = targetUI;
        this.resultCode = resultCode;
        this.requestCode = requestCode;
        this.data = data;
    }

    public int requestCode() {
        return this.requestCode;
    }

    public int resultCode() {
        return this.resultCode;
    }

    public Intent data() {
        return this.data;
    }

    public T targetUI() {
        return this.targetUI;
    }
}
```
## 总结
RxJava强大在于其操作符，如果我们能够合理利用操作符，我们的代码能够变得更加简洁。
