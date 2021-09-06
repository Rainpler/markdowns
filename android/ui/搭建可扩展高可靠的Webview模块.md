## 前言
WebView在我们的安卓开发过程中，是一个不可或缺的组件。通过WebView的使用，可以减少开发的流程，一套页面多个系统都可以使用，同时当我们的业务需求交互比较少，点击深度比较深的时候，使用WebView也可以提高App的性能，因此WebView的使用场景越来越广泛。

我们要实现的WebView应该是**高可靠的**，当webview出了问题不影响主进程，还是**可扩展的**，可以实现html页面与native的通信，同时是**模块化的**，以满足设计的重用。

## WebView 的使用
首先我们先来看一下WebView的简单使用，相信大家都或多或少使用过WebView组件用于加载H5页面。WebView是一个基于WebKit引擎、展现Web页面的控件，我们要使用它只需要简单的两步:
1. 在布局文件中添加WebView控件
2. 在代码中让WebView加载网页

以加载百度为例，我们来实现一个简单的WebView例子，使用一个WebViewActivity来统一管理WebView页面的加载。

layout_webview_activity布局文件：我们在布局里面放了一个标题和后退按钮，然后使用WebView填充满剩余的空间。

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="@dimen/action_bar_height"
            android:background="#00ffffff"
            android:id="@+id/action_bar"
            android:gravity="center_vertical">

            <ImageView
                android:id="@+id/back"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentLeft="true"
                android:layout_marginTop="5dp"
                android:layout_marginBottom="5dp"
                android:layout_marginLeft="5dp"
                android:background="?selectableItemBackground"
                android:padding="5dp"
                android:src="@mipmap/pc_left_arrow" />


            <TextView
                android:id="@+id/title"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_centerInParent="true"
                android:gravity="center"
                android:singleLine="true"
                android:textColor="@android:color/black"
                android:textSize="20sp" />

        </RelativeLayout>
        <WebView
            android:id="@+id/web_view"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"/>
```
然后是WebViewActivity的代码，在这里我们使用了DataBinding：
```java
public class WebViewActivity extends AppCompatActivity {
    private ActivityWebviewBinding mBinding;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_webview);
        mBinding.title.setText(getIntent().getStringExtra(Constants.TITLE));
        mBinding.actionBar.setVisibility(getIntent().getBooleanExtra(Constants.IS_SHOW_ACTION_BAR, true)? View.VISIBLE:View.GONE);
        mBinding.back.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                WebViewActivity.this.finish();
            }
        });
        mBinding.webView.setJavaScriptEnabled(true);
        mBinding.webView.loadUrl(getIntent().getSerializableExtra(Constants.URL));
    }

    public void updateTitle(String title){
        mBinding.title.setText(title);
    }

    public static void startWebViewActivity(Context context, String url, String title, boolean isShowActionBar) {
        if (context != null) {
            Intent intent = new Intent(context, WebViewActivity.class);
            intent.putExtra(Constants.TITLE, title);
            intent.putExtra(Constants.URL, url);
            intent.putExtra(Constants.IS_SHOW_ACTION_BAR, isShowActionBar);
            context.startActivity(intent);
        }
    }
}
```
然后在我们的MainActivity中，只需要这么调用就可以了。当然还需要在AndroidManifest.xml中添加一下    `<uses-permission android:name="android.permission.INTERNET"/>`网络权限。
```java
public void switchToBaidu(){
    String url ="https://www.baidu.com";
    WebViewActivity.startWebViewActivity(this,url,"百度",true);
}
```
运行程序效果如下：

![](../../res/webview打开百度.jpg)

<!-- native
用户体验 数据安全  流量转化 -->

<!-- #### WebChromeClient

#### WebViewClient

#### WebSettings -->

可能在我们很多人的开发中，都是这样简单的去使用WebView控件，但是这样会带来不少问题。首先，我们要实现的是一个WebView的模块，所以在MainActivity中不应该直接调用webview.class去打开WebView。其次，我们现在不能满足在Fragment中打开WebView的需求，所以应该对页面进行重构。同时，由于WebView与主进程同在一个进程内，而WebView又会占用较多的内存，当内存不足的时候就容易产生崩溃，因此为了提高可靠性，可以考虑跨进程实现。


### 模块化
所谓模块化，就是解决一个复杂问题时自顶向下逐层把系统划分成若干模块的过程。在我们的代码实现中，就是把WebView的使用抽离成为webview模块。在Android中，实现模块化需要我们新建一个Module，并在build.gradle中声明为`apply plugin: 'com.android.library'`，在settings.gradle中添加引用`include ':webview'`。当然如果使用Android Studio的话，ide会帮我们完成这一系列工作。

#### 层次化

<!-- 组件化 控件化 -->

组件化
arouter bug


autoservice google

接口下沉？ cc 根据字符串 没有编译校验
提供功能给组件用   组件之间相互不依赖

api  public
implementation  private

SmartRefreshLayout

#### LoadSir



#### JavascriptInterface

### 跨进程通信 Aidl
内存不少  jvm webviww
可靠性  不影响app进程  跨进程
webview崩溃 不影响app



违背开闭原则

主线程通信 aidl
