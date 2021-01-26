# Retrofit中的注解、反射与代理模式
Retrofit 是一个 RESTful 的 HTTP 网络请求框架的封装，网络请求的工作本质上是 OkHttp 完成，而 Retrofit 仅负责网络请求接口的封装，其内部实现实际上是使用了代理模式，为了更好的学习Retrofit框架，我们先从代理模式开始。
### 代理模式
>代理模式，就是为其他对象提供一种代理以控制对这个对象的访问。如果在直接访问对象时带来的问题，比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因（比如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问），直接访问会给使用者或者系统结构带来很多麻烦，我们可以在访问此对象时加上一个对此对象的访问层。
#### 代理模式的使用
来看一个代理模式的例子。

**定义接口：**
```java
/**
 * 代理抽象角色： 定义了服务的接口
 * 代理角色和真实角色对外提供的公共方法，一般为一个接口
 */
public interface Sing{
  public void sing();
}
```

**实现接口：**
```java
/**
 *  实现类： 提供唱歌服务的Jack
 *  定义了真是角色所要实现的业务逻辑
 */
 public class Jack implements Sing{
   @Override
   public void sing(){
     System.out.println("努力唱歌")
   }
 }

```
**代理对象：**
```java
/**
 * 代理对象：演唱代理人
 * 是真实角色的代理，通过真实角色的业务逻辑方法来实现抽象方法，并可以附加自己的操作
 * 将统一的流程控制都放到代理角色中处理。
 */
public class SingAgent implements Sing {

    private final Sing sing;

    public Agent(Sing Sing) {
        this.Sing = sing;
    }

    //....前置处理
    public void before() {
        System.out.println("准备");
    }

    //....后置处理
    public void after() {
        System.out.println("打分");
    }

    @Override
    public void massage() {
        before();
        Sing.sing();
        after();
    }
}
```
**代理模式使用：**
```java
public class Main {

    public static void main(String[] args) throws Exception {
        //静态代理
       Sing sing = new Jack();
       SingAgent agent = new Agent(sing);  
       agent.sing();
      }
}
```
在静态代理中，一个代理类可以代理多个真实对象，我们除了定义Jack之外，还可以定义David。如果我们的Jack除了会Sing（唱歌）之外，还会Dance(跳舞)。
```java
public interface Dance{
  public void dance();
}

public class Jack implements Sing, Dance{
  ......
}
```
但是一个代理，只能实现一个抽象接口，为了能代理Jack对象，必须再创建一个DanceAgent，这样一来，代码中就会有很多代理类。所以我们就必须想办法，通过一个代理类，实现全部的代理功能，这时候就用到了动态代理。
```java

public static void main(String[] args) throws Exception {
    //静态代理
   Jack jack = new Jack();

   Object o = Proxy.newProxyInstance(Main.class.getClassLoader(), //类加载器
           new Class[]{ Sing.class, Dance.class },  //代理的接口数组
           new InvocationHandler() {  //回调方法
               @Override
               public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
  //                        System.out.println(o.toString());
                   return method.invoke(jack, objects);
               }
           });
   Sing sing = (Sing) o;
   sing.sing();
  }

```
在这里，使用了``Proxy``类中的``newProxyInstance``方法，需要提供三个参数，类加载器、代理的接口数组、回调方法。先创建了一个Jack对象，然后创建了一个动态代理对象o，然后通过类型转换，将o转化成要代理的接口，调用``sing()``方法的时候，就会触发``InvocationHandler.invoke()``这个回调。回调方法中也有三个参数，o就是我们的动态对理对象，method就是接口的方法，objects就是方法参数。然后调用``method.invoke()``真正实现``jack``的``sing()``方法

所以，现在我们大概能了解到什么时候使用代理模式了吧。比如在早期网络实现的时候，用的是Volley框架，如果对于每一条网络请求，都是直接用Volley去建立，那么到了后面换成Okhttp框架的时候，就要对每一条网络请求都去修改代码，十分麻烦。而使用代理模式，则可以添加一个服务接口（定义了get和post），把Volley封装，并创建代理类，在代理对象中调用封装层的真实实现。后期切换框架的时候也更方便。
#### Proxy.newProxyInstance()原理
```Java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone(); //拷贝接口
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs); //获取Class对象

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams); //利用反射获取class 的构造方法
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});  //对class进行实例化
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```
在``newProxyInstance``方法中，``interfaces.clone()``先对接口数组进行了拷贝，然后利用``getProxyClass0``方法创建``Class``对象，然后通过反射，获得它的构造方法``cons``，最后利用``cons.newInstance()``创建该类的实例。

我们的类是怎么来的？我们先来回顾一下类的完整生命周期。

Java源文件(.java) —编译—> Java字节码(.class)  —类加载—> Class对象 —实例化—> 实例对象 ———>卸载

而我们的.class文件一般都是一个实实在在的文件，是在硬盘中存在的。但是在动态代理中，他这个Class对象，是在内存中去生成的。我们来看``getProxyClass0``这个方法
```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        return proxyClassCache.get(loader, interfaces);
    }
```
在``Proxy``里面有一个``proxyClassCache``对象，该对象是一个``WeakCache``实例，该缓存可以保存已经生成过的代理类，如果有则直接返回。如果没有的话，则通过``ProxyClassFactory``去创建代理类对象。
```Java
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        ......
        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            ......

            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {

                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```
``ProxyClassFactory``中，通过``ProxyGenerator.generateProxyClass``创建了代理类数据，返回的是一个byte数组，然后通过``defineClass0``去解析这个byte数组，并生成一个代理类对象。我们可以把这个byte数组通过数据流的方式输出到文件中。
```java
private static void proxy() throws Exception {
       String name = Sing.class.getName() + "$Proxy0";
       //生成代理指定接口的Class数据
       byte[] bytes = ProxyGenerator.generateProxyClass(name, new Class[]{Sing.class});
       FileOutputStream fos = new FileOutputStream("lib/" + name + ".class");
       fos.write(bytes);
       fos.close();
   }
```
我们来观察它的文件结构。
```java
public final class Sing$Proxy0 extends Proxy implements Sing {
    private static Method m3;

    //构造方法接受InvocationHandler
    public Sing$Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }
    ......

    //在super中，也就是Proxy中，h就是传入的InvocationHandler
    public final void sing() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
    ......

    //静态代码块中，利用反射，获得sing()方法
    static {
            try {
                m3 = Class.forName("com.enjoy.lib.Sing").getMethod("sing");
            } catch (NoSuchMethodException var2) {
                throw new NoSuchMethodError(var2.getMessage());
            } catch (ClassNotFoundException var3) {
                throw new NoClassDefFoundError(var3.getMessage());
            }
        }
}
```

## Retrofit的简单实现
在接下来，我们将利用注解、反射与动态代理，对Retrofit进行一个简单的实现。在此之前，我们先来看一下Retrofit的简单使用。
#### Retrofit的基本使用
**Api定义：**
创建接api接口
```java
//这里使用的是高德地图提供的天气api
public interface WeatherApi {

    @POST("/v3/weather/weatherInfo")
    @FormUrlEncoded
    Call<ResponseBody> postWeather(@Field("city") String city, @Field("key") String key);


    @GET("/v3/weather/weatherInfo")
    Call<ResponseBody> getWeather(@Query("city") String city, @Query("key") String key);
}

```
**Retrofit使用：**
创建Retrofit对象，然后利用``create()``创建了api接口的对象，最后调用api的方法。
```java
Retrofit retrofit = new Retrofit.Builder().baseUrl("https://restapi.amap.com")
               .build();
weatherApi = retrofit.create(WeatherApi.class);
Call<ResponseBody> call = weatherApi.getWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
        call.enqueue(new Callback<ResponseBody>() {
            @Override
            public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {
                if (response.isSuccessful()){
                    ResponseBody body = response.body();
                    try {
                        String string = body.string();
                        Log.i(TAG, "onResponse get: " + string);
                    } catch (IOException e) {
                        e.printStackTrace();
                    } finally {
                        body.close();
                    }
                }

            }
            @Override
            public void onFailure(Call<ResponseBody> call, Throwable t) {

            }
        });
```
观察上面代码，Retrofit在创建的时候，使用的是构建者模式，它可以将一个复杂对象的构建和它的表示分离，可以让使用者方便使用，不必知道内部的细节。而在创建Api接口对象的时候， 使用的就是动态代理。接下来我们来实现一个自己的Retrofit。

**EnjoyRetrofit.java**
```java
public class EnjoyRetrofit {

    final Map<Method, ServiceMethod> serviceMethodCache = new ConcurrentHashMap<>();
    final Call.Factory callFactory;
    final HttpUrl baseUrl;

    EnjoyRetrofit(Call.Factory callFactory, HttpUrl baseUrl) {
        this.callFactory = callFactory;
        this.baseUrl = baseUrl;
    }

    public <T> T create(final Class<T> service) {
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        //解析这个method 上所有的注解信息
                        ServiceMethod serviceMethod = loadServiceMethod(method);
                        //args:
                        return serviceMethod.invoke(args);
                    }
                });
    }

    private ServiceMethod loadServiceMethod(Method method) {
        //先不上锁，避免synchronized的性能损失
        ServiceMethod result = serviceMethodCache.get(method);
        if (result != null) return result;
        //多线程下，避免重复解析,
        synchronized (serviceMethodCache) {
            result = serviceMethodCache.get(method);
            if (result == null) {
                result = new ServiceMethod.Builder(this, method).build();
                serviceMethodCache.put(method, result);
            }
        }
        return result;
    }


    /**
     * 构建者模式，将一个复杂对象的构建和它的表示分离，可以使使用者不必知道内部组成的细节。
     */
    public static final class Builder {
        private HttpUrl baseUrl;
        //Okhttp->OkhttClient
        private okhttp3.Call.Factory callFactory;  //null


        public Builder callFactory(okhttp3.Call.Factory factory) {
            this.callFactory = factory;
            return this;
        }

        public Builder baseUrl(String baseUrl) {
            this.baseUrl = HttpUrl.get(baseUrl);
            return this;
        }

        public EnjoyRetrofit build() {
            if (baseUrl == null) {
                throw new IllegalStateException("Base URL required.");
            }
            okhttp3.Call.Factory callFactory = this.callFactory;
            if (callFactory == null) {
                callFactory = new OkHttpClient();
            }

            return new EnjoyRetrofit(callFactory, baseUrl);
        }
    }
}

```
**ServiceMethod.java**
```java
/**
 * 记录请求类型  参数  完整地址
 */
public class ServiceMethod {

    private final Call.Factory callFactory;
    private final String relativeUrl;
    private final boolean hasBody;
    private final ParameterHandler[] parameterHandler;
    private FormBody.Builder formBuild;
    HttpUrl baseUrl;
    String httpMethod;
    HttpUrl.Builder urlBuilder;

    public ServiceMethod(Builder builder) {
        baseUrl = builder.enjoyRetrofit.baseUrl;
        callFactory = builder.enjoyRetrofit.callFactory;

        httpMethod = builder.httpMethod;
        relativeUrl = builder.relativeUrl;
        hasBody = builder.hasBody;
        parameterHandler = builder.parameterHandler;

        //如果是有请求体,创建一个okhttp的请求体对象
        if (hasBody) {
            formBuild = new FormBody.Builder();
        }
    }

    public Object invoke(Object[] args) {
        /**
         * 1  处理请求的地址与参数
         */
        for (int i = 0; i < parameterHandler.length; i++) {
            ParameterHandler handlers = parameterHandler[i];
            //handler内本来就记录了key,现在给到对应的value
            handlers.apply(this, args[i].toString());
        }

        //获取最终请求地址
        HttpUrl url;
        if (urlBuilder == null) {
            urlBuilder = baseUrl.newBuilder(relativeUrl);
        }
        url = urlBuilder.build();

        //请求体
        FormBody formBody = null;
        if (formBuild != null) {
            formBody = formBuild.build();
        }

        Request request = new Request.Builder().url(url).method(httpMethod, formBody).build();
        return callFactory.newCall(request);
    }

    // get请求,  把 k-v 拼到url里面
    public void addQueryParameter(String key, String value) {
        if (urlBuilder == null) {
            urlBuilder = baseUrl.newBuilder(relativeUrl);
        }
        urlBuilder.addQueryParameter(key, value);
    }

    //Post   把k-v 放到 请求体中
    public void addFiledParameter(String key, String value) {
        formBuild.add(key, value);
    }


    public static class Builder {

        private final EnjoyRetrofit enjoyRetrofit;
        private final Annotation[] methodAnnotations;
        private final Annotation[][] parameterAnnotations;
        ParameterHandler[] parameterHandler;
        private String httpMethod;
        private String relativeUrl;
        private boolean hasBody;

        public Builder(EnjoyRetrofit enjoyRetrofit, Method method) {
            this.enjoyRetrofit = enjoyRetrofit;
            //获取方法上的所有的注解
            methodAnnotations = method.getAnnotations();
            //获得方法参数的所有的注解 (一个参数可以有多个注解,一个方法又会有多个参数)
            parameterAnnotations = method.getParameterAnnotations();
        }

        public ServiceMethod build() {

            /**
             * 1 解析方法上的注解, 只处理POST与GET
             */
            for (Annotation methodAnnotation : methodAnnotations) {
                if (methodAnnotation instanceof POST) {
                    //记录当前请求方式
                    this.httpMethod = "POST";
                    //记录请求url的path
                    this.relativeUrl = ((POST) methodAnnotation).value();
                    // 是否有请求体
                    this.hasBody = true;
                } else if (methodAnnotation instanceof GET) {
                    this.httpMethod = "GET";
                    this.relativeUrl = ((GET) methodAnnotation).value();
                    this.hasBody = false;
                }
            }


            /**
             * 2 解析方法参数的注解
             */
            int length = parameterAnnotations.length;
            parameterHandler = new ParameterHandler[length];
            for (int i = 0; i < length; i++) {
                // 一个参数上的所有的注解
                Annotation[] annotations = parameterAnnotations[i];
                // 处理参数上的每一个注解
                for (Annotation annotation : annotations) {
                    //todo 可以加一个判断:如果httpMethod是get请求,现在又解析到Filed注解,可以提示使用者使用Query注解
                    if (annotation instanceof Field) {
                        //得到注解上的value: 请求参数的key
                        String value = ((Field) annotation).value();
                        parameterHandler[i] = new ParameterHandler.FiledParameterHandler(value);
                    } else if (annotation instanceof Query) {
                        String value = ((Query) annotation).value();
                        parameterHandler[i] = new ParameterHandler.QueryParameterHandler(value);

                    }
                }
            }

            return new ServiceMethod(this);
        }
    }
}

```
**ParameterHandler.java**
```java
public abstract class ParameterHandler {

    abstract void apply(ServiceMethod serviceMethod, String value);


    static class QueryParameterHandler extends ParameterHandler {
        String key;

        public QueryParameterHandler(String key) {
            this.key = key;
        }

        //serviceMethod: 回调
        @Override
        void apply(ServiceMethod serviceMethod, String value) {
            serviceMethod.addQueryParameter(key,value);
        }
    }

    static class FiledParameterHandler extends ParameterHandler {
        String key;

        public FiledParameterHandler(String key) {
            this.key = key;
        }

        @Override
        void apply(ServiceMethod serviceMethod, String value) {
            serviceMethod.addFiledParameter(key,value);
        }
    }
}
```
**定义注解：**
Field.java
```java
@Target(PARAMETER)
@Retention(RUNTIME)
public @interface Field {

    String value();
}
```
GET.java
```java
@Target(METHOD)
@Retention(RUNTIME)
public @interface GET {

    String value() default "";
}
```
POST.java
```java
@Target(METHOD)
@Retention(RUNTIME)
public @interface POST {

    String value() default "";
}
```
Query.java
```java
@Target(PARAMETER)
@Retention(RUNTIME)
public @interface Query {

    String value();
}
```
最后，我们来看如何使用自定义的Retrofit。
```java
public class MainActivity extends AppCompatActivity {

    private WeatherApi weatherApi;
    private static final String TAG = "MainActivity";
    private EnjoyWeatherApi enjoyWeatherApi;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        EnjoyRetrofit enjoyRetrofit = new EnjoyRetrofit.Builder().baseUrl("https://restapi.amap.com").build();
        enjoyWeatherApi = enjoyRetrofit.create(EnjoyWeatherApi.class);
    }

    public void enjoyGet(View view) {
        okhttp3.Call call = enjoyWeatherApi.getWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
        call.enqueue(new okhttp3.Callback() {
            @Override
            public void onFailure(okhttp3.Call call, IOException e) {

            }

            @Override
            public void onResponse(okhttp3.Call call, okhttp3.Response response) throws IOException {
                Log.i(TAG, "onResponse enjoy get: " + response.body().string());
                response.close();
            }
        });

    }

    public void enjoyPost(View view) {
        okhttp3.Call call = enjoyWeatherApi.postWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
        call.enqueue(new okhttp3.Callback() {
            @Override
            public void onFailure(okhttp3.Call call, IOException e) {

            }

            @Override
            public void onResponse(okhttp3.Call call, okhttp3.Response response) throws IOException {
                Log.i(TAG, "onResponse enjoy post: " + response.body().string());
                response.close();
            }
        });
    }
}
```
