## 标准观察者设计模式
RxJava是一种特殊的观察者模式，首先我们先来看标准的观察者设计模式。在标准观察者模式中，存在两种对象，一种是观察者，一种是被观察者，“被观察者与“观察者之间是一对多的关系。当被观察者发出通知改变的时候，观察者才能察觉到。

**Observerable.java**
```java
public interface Observerable {
    private List<Observer> observers = new ArrayList<>();

    public void addObserver(Observer observer) ;

    public void removeObserver(Observer observer);

    public void notifyObservers() ;

    public void pushMessage(String msg);
}
```

**Observer.java**
```java
public interface Observer{
    void update();
}
```

## RxJava Hook（钩子）
>Hook技术又叫钩子函数，在系统没有调用函数之前，钩子就先捕获该消息，得到控制权。这时候钩子程序既可以改变该程序的执行，插入我们要执行的代码片段，还可以强制结束消息的传递。


**RxJava中的hook的使用**
```java
RxJavaPlugins.setOnObservableAssembly(new Function<Observable, Observable>() {
        @Override
        public Observable apply(Observable observable) throws Throwable {
            return observable;
        }
    });
```
来观察这么一个代码段：
```java
public class MainActivity extends AppCompatActivity {

    @InjectView(R.id.tv_text)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        RxJavaPlugins.setOnObservableAssembly(new Function<Observable, Observable>() {
            @Override
            public Observable apply(Observable observable) throws Throwable {
                System.out.println("apply : " + observable);
                return observable;
            }
        });
        testHook();
    }

    private void testHook() {
        Observable.create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Object> e) throws Throwable {
                e.onNext(null);
            }
        }).map(new Function<Object, Boolean>() {
            @Override
            public Boolean apply(Object o) throws Throwable {
                return null;
            }
        }).subscribe(subscribe(new Observer<Boolean>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {

            }

            @Override
            public void onNext(@NonNull Boolean aBoolean) {

            }

            @Override
            public void onError(@NonNull Throwable e) {

            }

            @Override
            public void onComplete() {

            }
        });
    }
}
```

运行后``Logcat``打印出
```java
2021-01-18 14:18:53.452 7793-7793/com.example.anatationtest I/System.out: apply : io.reactivex.rxjava3.internal.operators.observable.ObservableCreate@1569e83
2021-01-18 14:18:53.452 7793-7793/com.example.anatationtest I/System.out: apply : io.reactivex.rxjava3.internal.operators.observable.ObservableMap@27f9800
```
可以看到``ObservableCreate``和``ObservableMap``都被hook方法捕获了，这样就可以实现RxJava的全局监听。观察``create()``源码：

```java
public static <T> Observable<T> create(@NonNull ObservableOnSubscribe<T> source) {
    Objects.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<>(source));
}
```
和``map()``方法源码：

```java
public final <R> Observable<R> map(@NonNull Function<? super T, ? extends R> mapper) {
      Objects.requireNonNull(mapper, "mapper is null");
      return RxJavaPlugins.onAssembly(new ObservableMap<>(this, mapper));
  }
```
我们再看``RxJavaPlugins.onAssembly()``方法：
```java
@NonNull
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```
在这里有个``onObservableAssembly``，如果该变量不为空，则为``Observable``对象应用``apply(f,source)``，再往前寻找，找到该变量被赋值的地方，发现就是在我们前面通过``setOnObservableAssembly()``函数设置全局监听，顾名思义，就是设置被观察者的装配。
```java
public static void setOnObservableAssembly(@Nullable Function<? super Observable, ? extends Observable> onObservableAssembly) {
      if (lockdown) {
          throw new IllegalStateException("Plugins can't be changed anymore");
      }
      RxJavaPlugins.onObservableAssembly = onObservableAssembly;
  }
```

## RxJava 观察者设计模式
#### 两者观察者设计模式对比
在标准观察者设计模式中，“被观察者”与“观察者”是一对多的关系，并且需要被观察者发出改变通知后，所有的观察者才能观察到。在RxJava观察者设计模式中，“被观察者”与“观察者”是多对一的关系，并且需要在起点和终点订阅一次后，才能发出改变通知，也可以称之为发布订阅模式。

在标准观察者模式中，被观察者中的容器直接存放观察者的引用，耦合度更高。RxJava中，被观察者与观察者通过抽象层发射器连接，降低了耦合度。

1. Observer源码
2. Observerable创建过程，源码分析
3. subscribe订阅过程，源码分析
4. map转换流程，源码分析

### Observer

```java
//Observer是一个泛型接口，包含四个抽象方法
public interface Observer<@NonNull T> {

     //当Observer对象被Observerable对象subscribe时候马上调用
     //d 为此次订阅的事件对象，如果调用dispose，则订阅会被杀死
    void onSubscribe(@NonNull Disposable d);

     //把被观察者从上游传递来的对象给观察者
    void onNext(@NonNull T t);

     //通知观察者，被观察者发送了错误，调用此方法将不会调用onNext()和onComplete()方法
    void onError(@NonNull Throwable e);

    //通知观察者，被观察者已经发送完事件，如果抛出onError()错误则不会调用
    void onComplete();

}
```

### Observerable创建过程

首先调用``create()``方法，传入自定义source作为参数

```java
public static <T> Observable<T> create(@NonNull ObservableOnSubscribe<T> source) {
        Objects.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<>(source));
    }
```
接着先对传进来的source进行判空，如果为空，则直接抛出异常。如果不为空，则走``onAssembly()``方法，该方法在上面已经分析过了，所以我们再来看``new ObservebleCreate<>(source)``的创建过程。
```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
    ......
}
```
``ObservableCreate``中创建了一个``source``成员变量，并在创建的时候直接将传入的自定义source赋值给它。
### subscribe订阅过程
```java
observable.subscribe(new Observer(object){
  ......
})
```
我们来分析``subcribe()``函数

```java
public final void subscribe(@NonNull Observer<? super T> observer) {
    Objects.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);
        Objects.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
      //错误处理
      ......
    }

    protected abstract void subscribeActual(@NonNull Observer<? super T> observer);

}
```
前面几行代码是对``observer``对象进行校验，如果校验没问题，最后调用``subscribeActual()``方法。而该方法是一个抽象方法，那么是谁实现了该方法呢，是我们的``ObservableCreate``，因此我们再来看它里面``subscribeActual()``的实现。
```java
@Override
   protected void subscribeActual(Observer<? super T> observer) {
       CreateEmitter<T> parent = new CreateEmitter<>(observer);
       observer.onSubscribe(parent);

       try {
           source.subscribe(parent);
       } catch (Throwable ex) {
           Exceptions.throwIfFatal(ex);
           parent.onError(ex);
       }
   }
```
首先``CreateEmitter``为我们传进来的自定义``observer``创建一个事件发射，并将它包裹起来，然后调用``observer.onSubscribe()``，这就是前文提到一调用``subscribe()``，``observer.onSubscribe()``就会调用的原因。

接着调用``source.subcribe()``，把创建的事件发射器传进去。还记得source是什么吗，它就是我们一开始创建的自定义source，而该方法的抽象实现又回到了在我们代码最开始的地方。
```java
Observable.create(new ObservableOnSubscribe<Object>() {
         @Override
         public void subscribe(@NonNull ObservableEmitter<Object> e) throws Throwable {
             e.onNext("");
         }
     })
```
然后就会调用到``CreateEmitter.onNext()``
```java
public void onNext(T t) {
      if (t == null) {
          onError(ExceptionHelper.createNullPointerException("onNext called with a null value."));
          return;
      }
      if (!isDisposed()) {
          observer.onNext(t);
      }
  }
```
由于``CreateEmitter``中已经持有了``observer``的对象，最后就会调用到``observer.onNext(t)``，这样完整的订阅流程就形成了。

### map()源码分析
```java
public final <R> Observable<R> map(@NonNull Function<? super T, ? extends R> mapper) {
      Objects.requireNonNull(mapper, "mapper is null");
      return RxJavaPlugins.onAssembly(new ObservableMap<>(this, mapper));
  }
```
我们重点来``ObservableMap``这个类。
```java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        //保存function
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        ......
    }
}
```
首先在构造函数中，将Function保存起来，``map()``之后，返回的依旧是``Observerable``对象，并将``ObserverableCreate``对象保存为它的source，接下来会走到``ObservableMap.subscribeActual()``中，然后为``observer``对象`t`包裹一层``MapObserver``。这时候再调用``ObserverableCreate.subscribeActual()``,后续的订阅流程就跟之前一样了。

在这里，我们重点分析``MapObserver``中的处理。
```java
static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
  final Function<? super T, ? extends U> mapper;

  MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
      super(actual);
      this.mapper = mapper;
  }

  @Override
  public void onNext(T t) {
      if (done) {
          return;
      }

      if (sourceMode != NONE) {
          downstream.onNext(null);
          return;
      }

      U v;

      //mapper合法性校验
      try {
         //应用function转换
          v = Objects.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
      } catch (Throwable ex) {
          fail(ex);
          return;
      }
      //向下游传递
      //实际为我们自定义的observer
      downstream.onNext(v);

      @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        //异常处理
        @Nullable
        @Override
        public U poll() throws Throwable {
            T t = qd.poll();
            return t != null ? Objects.requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
  }
}
```
由前面的流程分析，会一直调用到``CreateEmitter.onNext()``，然后又会进一步对``M apObserver``进行拆包，进而走到它的``MapObserver.onNext()``方法中，先对Function 对象 ``mapper``进行合法性校验，然后调用``apply()``函数，这里的apply函数是抽象函数。具体实现在我们的链式调用中，最后得到返回值v，也就是经过变换后的对象，并调用``downstream.onNext(v)`` 向下游传递。``downsteam``则为被包裹住的``observer``对象。
```java
new Function<Object, Boolean>() {
    @Override
    public Boolean apply(Object o) throws Throwable {
        return null;
    }
}
```
为什么RxJava要这么写呢，从上文的分析中，我们可以看到在链式调用中，如果调用多次``map()``，相当于为自定义observer封装了层层包裹，发布订阅的过程就相当于是封包->拆包的过程，代码逻辑清晰，避免了函数嵌套。这就是一种装饰模型，一开始我们创建了``ObserverableCreate``，然后为它穿上一件件``ObserverableMap``的外套，然后``subscribe``的过程就是脱外套的过程。
## 总结
本篇文章通过对RxJava中Observer、Observable、subscribe源码分析，比较了其与标准观察者设计模式的差别，更深的学习了RxJava的思想和原理。
