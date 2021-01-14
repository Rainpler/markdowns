# RxJava 应用与原理
>RxJava 在 GitHub 主页上的自我介绍是 "a library for composing asynchronous and event-based programs using observable sequences for the Java VM"（一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库）。这就是 RxJava ，概括得非常精准。

生活中的例子：

起点（分发事件(PATH)：我饿了）----------下楼-------去餐厅--------点餐----------> 终点（吃饭 消费事件）

程序中的例子：
gg
起点（分发事件：点击登录）----------登录API-------请求服务器--------获取响应码----------> 终点（更新UI登录成功 消费事件）

我们想要吃饭，必须先点餐，想点餐，必须先去餐厅，只有前面的事情做完了，后面的事情才能做，那么这种就称为驱动式事件。RxJava 是响应式编程的一种解决方式。

本文主要包括以下几个部分：
1. RxJava 应用场景
2. Rxjava配合Retrofit
3. 防抖
4. doOnNext

## RxJava 应用场景


```java
    //打印logcat日志的标签
    private static final String TAG = DownloadActivity.class.getSimpleName();

    // 网络图片的链接地址
    private final static String PATH = "http://pic1.win4000.com/wallpaper/c/53cdd1f7c1f21.jpg";

    public void downloadImageAction(View view) {
        progressDialog = new ProgressDialog(this);
        progressDialog.setTitle("下载图片中...");
        progressDialog.show();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    URL url = new URL(PATH);
                    HttpURLConnection httpURLConnection = (HttpURLConnection) url.openConnection();
                    httpURLConnection.setConnectTimeout(5000);
                    int responseCode = httpURLConnection.getResponseCode(); // 才开始 request
                    if (responseCode == HttpURLConnection.HTTP_OK) {
                        InputStream inputStream = httpURLConnection.getInputStream();
                        Bitmap bitmap = BitmapFactory.decodeStream(inputStream);
                        Message message = handler.obtainMessage();
                        message.obj = bitmap;
                        handler.sendMessage(message);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

```

```java
public void rxJavaDownloadImageAction(View view) {
        // 起点
        Observable.just(PATH)  // 内部会分发  PATH Stirng  // TODO 第二步

         // TODO 第三步
        .map(new Function<String, Bitmap>() {
            @Override
            public Bitmap apply(String s) throws Exception {
                URL url = new URL(PATH);
                HttpURLConnection httpURLConnection = (HttpURLConnection) url.openConnection();
                httpURLConnection.setConnectTimeout(5000);
                int responseCode = httpURLConnection.getResponseCode(); // 才开始 request
                if (responseCode == HttpURLConnection.HTTP_OK) {
                    InputStream inputStream = httpURLConnection.getInputStream();
                    Bitmap bitmap = BitmapFactory.decodeStream(inputStream);
                    return bitmap;
                }
                return null;
            }
        })

        .subscribeOn(Schedulers.io())     // 给上面代码分配异步线程
        .observeOn(AndroidSchedulers.mainThread())// 给下面代码分配主线程;
        //.compose(rxud())
        // 订阅 起点 和 终点 订阅起来
        .subscribe(

                // 终点
                new Observer<Bitmap>() {

                    // 订阅开始
                    @Override
                    public void onSubscribe(Disposable d) {
                        // 预备 开始 要分发
                        // TODO 第一步
                        progressDialog = new ProgressDialog(DownloadActivity.this);
                        progressDialog.setTitle("download run");
                        progressDialog.show();
                    }

                    // TODO 第四步
                    // 拿到事件
                    @Override
                    public void onNext(Bitmap bitmap) {
                        image.setImageBitmap(bitmap);
                    }

                    // 错误事件
                    @Override
                    public void onError(Throwable e) {

                    }

                    // TODO 第五步
                    // 完成事件
                    @Override
                    public void onComplete() {
                        if (progressDialog != null)
                            progressDialog.dismiss();
                    }
        });

    }
```
得益于RxJava的链式编程，增加需求变得简洁明了，假如有一个需求需要将图片再加上水印、并添加日志记录，那么可以在链的后面再添加两个map操作方法
```java
...
// 图片上绘制文字 加水印
.map(new Function<Bitmap, Bitmap>() {
    @Override
    public Bitmap apply(Bitmap bitmap) throws Exception {
        Paint paint = new Paint();
        paint.setTextSize(88);
        paint.setColor(Color.RED);
        Bitmap.Config bitmapConfig = bitmap.getConfig();

        paint.setDither(true); // 获取跟清晰的图像采样
        paint.setFilterBitmap(true);// 过滤一些
        if (bitmapConfig == null) {
            bitmapConfig = Bitmap.Config.ARGB_8888;
        }
        bitmap = bitmap.copy(bitmapConfig, true);
        Canvas canvas = new Canvas(bitmap);

        canvas.drawText(text, paddingLeft, paddingTop, paint);
        return bitmap;
    }
})

// 日志记录
.map(new Function<Bitmap, Bitmap>() {
    @Override
    public Bitmap apply(Bitmap bitmap) throws Exception {
        Log.d(TAG, "apply: 是这个时候下载了图片啊:" + System.currentTimeMillis());
        return bitmap;
    }
})
...

```

还可以将线程分配的任务封装起来
```java
/**
 * 封装我们的操作
 * UD
 * upstream   上游
 * downstream 下游
 */
public final static <UD> ObservableTransformer<UD, UD> rxud() {
    return new ObservableTransformer<UD, UD>() {
        @Override
        public ObservableSource<UD> apply(Observable<UD> upstream) {
            return  upstream.subscribeOn(Schedulers.io())     // 给上面代码分配异步线程
            .observeOn(AndroidSchedulers.mainThread()) // 给下面代码分配主线程;
            .map(new Function<UD, UD>() {
                @Override
                public UD apply(UD ud) throws Exception {
                    Log.d(TAG, "apply: 我监听到你了，居然再执行");
                    return ud;
                }
            });
            // .....        ;
        }
    };
}
```
## RxJava配合Retrofit
>Retrofit 是一个 RESTful 的 HTTP 网络请求框架的封装。网络请求的工作本质上是 OkHttp 完成，而 Retrofit 仅负责 网络请求接口的封装

Retrofit 除了提供了传统的 Callback 形式的 API，还有 RxJava 版本的 Observable 形式 API。这里使用一个获取玩安卓Api来作为例子讲解
**WanAndroidApi**
```java
public interface WanAndroidApi {

    // 总数据
    @GET("project/tree/json")
    Observable<ProjectBean> getProject();  // 异步线程 耗时操作

    // ITem数据
    @GET("project/list/{pageIndex}/json") // ?cid=294
    Observable<ProjectItem> getProjectItem(@Path("pageIndex") int pageIndex, @Query("cid") int cid);  // 异步线程 耗时操作
}
```
**Retrofit**
```java
public static String BASE_URL = "https://www.wanandroid.com/";

public static void setBaseUrl(String baseUrl) {
    BASE_URL = baseUrl;
}

/**
 * 根据各种配置创建出Retrofit
 *
 * @return 返回创建好的Retrofit
 */
public static Retrofit getOnlineCookieRetrofit() {
    // OKHttp客户端
    OkHttpClient.Builder httpBuilder = new OkHttpClient.Builder();
    // 各种参数配置
    OkHttpClient okHttpClient = httpBuilder
            .addNetworkInterceptor(new StethoInterceptor())
            .readTimeout(10000, TimeUnit.SECONDS)
            .connectTimeout(10000, TimeUnit.SECONDS)
            .writeTimeout(10000, TimeUnit.SECONDS)
            .build();


    return new Retrofit.Builder().baseUrl(BASE_URL)
            // 请求用 OKhttp
            .client(okHttpClient)
            //响应RxJava
            // 添加一个json解析的工具
            .addConverterFactory(GsonConverterFactory.create(new Gson()))
            // 添加rxjava处理工具
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())

            .build();
}
```

**在Activity中使用**
```java
/**
 * TODO Retrofit+RxJava 查询 项目分类  (总数据查询)
 *
 * @param view
 */
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    api = HttpUtil.getOnlineCookieRetrofit().create(WangAndroidApi.class);

}

public void getProjectAction(View view) {
    // 获取网络API
    api.getProject()
            .subscribeOn(Schedulers.io()) // 上面 异步
            .observeOn(AndroidSchedulers.mainThread()) // 下面 主线程
            .subscribe(new Consumer<ProjectBean>() {
                @Override
                public void accept(ProjectBean projectBean) throws Exception {
                    Log.d(TAG, "accept: " + projectBean); // UI 可以做事情
                }
            });
}
```
## 功能防抖
>防抖：一个函数连续多次触发，我们只执行最后一次。考虑这样一个场景：一个按钮被点击时，会发送网络请求。为了防止用户无意多次点击，或有人恶意连续发送请求，我们不希望按钮连续被点击时，每次都发送网络请求。而是过一定时间没有再点击时，我们才发送请求。即只执行最后一次，这便是防抖。

在RxJava家族中，有诸如RxJava、RxJs、RxBinding等等框架，这里为了防止控件抖动，可以使用``RxView.clicks()``来解决。
这里有一个点击按钮，先获取项目列表数据，再根据数据id获取项目item详情。
```java
@SuppressLint("CheckResult")
private void antiShakeActon() {
    // 注意：（项目分类）查询的id，通过此id再去查询(项目列表数据)

    // 对那个控件防抖动？
    Button bt_anti_shake = findViewById(R.id.bt_anti_shake);

    RxView.clicks(bt_anti_shake)
            .throttleFirst(2000, TimeUnit.MILLISECONDS) // 2秒钟之内 响应你一次
            .subscribe(new Consumer<Object>() {
                @Override
                public void accept(Object o) throws Exception {
                    api.getProject() // 查询主数据
                    .compose(DownloadActivity.rxud())
                    .subscribe(new Consumer<ProjectBean>() {
                        @Override
                        public void accept(ProjectBean projectBean) throws Exception {
                            for (ProjectBean.DataBean dataBean : projectBean.getData()) { // 10
                                // 查询item数据
                                api.getProjectItem(1, dataBean.getId())
                                .compose(DownloadActivity.rxud())
                                .subscribe(new Consumer<ProjectItem>() {
                                    @Override
                                    public void accept(ProjectItem projectItem) throws Exception {
                                        Log.d(TAG, "accept: " + projectItem); // 可以UI操作
                                    }
                                });
                            }
                        }
                    });
                }
            });
}
```
从上面的代码中可以看出，使用RxJava虽然实现了链式调用，但没有解决网络嵌套的问题，为此，我们使用``flatMap``操作符解决嵌套问题
```java
@SuppressLint("CheckResult")
private void antiShakeActonUpdate() {
    // 注意：项目分类查询的id，通过此id再去查询(项目列表数据)

    // 对那个控件防抖动？
    Button bt_anti_shake = findViewById(R.id.bt_anti_shake);

    RxView.clicks(bt_anti_shake)
            .throttleFirst(2000, TimeUnit.MILLISECONDS) // 2秒钟之内 响应你一次

            // 我只给下面 切换 异步
            .observeOn(Schedulers.io())
            .flatMap(new Function<Object, ObservableSource<ProjectBean>>() {
                @Override
                public ObservableSource<ProjectBean> apply(Object o) throws Exception {
                    return api.getProject(); // 主数据
                }
            })
            .flatMap(new Function<ProjectBean, ObservableSource<ProjectBean.DataBean>>() {
                @Override
                public ObservableSource<ProjectBean.DataBean> apply(ProjectBean projectBean) throws Exception {
                    return Observable.fromIterable(projectBean.getData()); // 我自己搞一个发射器 发多次 10 等同于上面的for循环
                }
            })
            .flatMap(new Function<ProjectBean.DataBean, ObservableSource<ProjectItem>>() {
                @Override
                public ObservableSource<ProjectItem> apply(ProjectBean.DataBean dataBean) throws Exception {
                    return api.getProjectItem(1, dataBean.getId());
                }
            })

            .observeOn(AndroidSchedulers.mainThread()) // 如果我要更新UI  给下面切换 主线程
            .subscribe(new Consumer<ProjectItem>() {
                @Override
                public void accept(ProjectItem projectItem) throws Exception {
                    Log.d(TAG, "accept: " + projectItem);
                }
            });
}

```
## doOnNext
> 我们先来看一个需求，请求服务器注册(耗时操作)->更新注册UI(mainThread)->请求服务器登录(耗时操作)->更新注册UI(UI线程)，在这样的业务中，线程频繁的发生切换，doOnNext就能很好的处理这样的需求。

**定义Api**
```java
// 请求接口 API
public interface Api {

    // 请求注册 功能  todo 耗时操作 ---> OkHttp
    public Observable<RegisterResponse> registerAction(@Body RegisterRequest registerRequest);

    // 请求登录 功能 todo 耗时操作 ---> OKHttp
    public Observable<LoginResponse> loginAction(@Body LoginRequest loginRequest);

}
```

**请求网络**

需求：
1. 请求服务器注册操作
2. 注册完成之后，更新注册UI
3. 马上去登录服务器操作
4. 登录完成之后，更新登录的UI


```java
private ProgressDialog progressDialog;

Disposable disposable;

public void request(View view) {
      Api api = retrofit.createRetrofit().create(Api.class);
      api.registerAction(new RegisterRequest()) // todo 1.请求服务器注册操作   // todo 2
      .subscribeOn(Schedulers.io()) // 给上面 异步
      .observeOn(AndroidSchedulers.mainThread()) // 给下面分配主线程
      .doOnNext(new Consumer<RegisterResponse>() { // todo 3
          @Override
          public void accept(RegisterResponse registerResponse) throws Exception {
              // todo 2.注册完成之后，更新注册UI
          }
      })
      // todo 3.马上去登录服务器操作
      .observeOn(Schedulers.io()) // 给下面分配了异步线程
      .flatMap(new Function<RegisterResponse, ObservableSource<LoginResponse>>() { // todo 4
          @Override
          public ObservableSource<LoginResponse> apply(RegisterResponse registerResponse) throws Exception {
              Observable<LoginResponse> loginResponseObservable =api.loginAction(new LoginReqeust());
              return loginResponseObservable;
          }
      })
      .observeOn(AndroidSchedulers.mainThread()) // 给下面 执行主线程
      .subscribe(new Observer<LoginResponse>() {

          // 一定是主线程，为什么，因为 subscribe 马上调用onSubscribe
          @Override
          public void onSubscribe(Disposable d) {
              // 1
              progressDialog = new ProgressDialog(RequestActivity.this);
              progressDialog.show();

              // UI 操作
              disposable = d;
          }

          @Override
          public void onNext(LoginResponse loginResponse) { // todo 5
              // 4.登录完成之后，更新登录的UI
          }

          @Override
          public void onError(Throwable e) {

          }

          // todo 6
          @Override
          public void onComplete() {
              if (progressDialog != null) {
                  progressDialog.dismiss();
              }
          }
      });

}

@Override
protected void onDestroy() {
    super.onDestroy();
    // 必须这样写，最起码的标准
    if (disposable != null)
        if (!disposable.isDisposed())
            disposable.dispose();
}
```
