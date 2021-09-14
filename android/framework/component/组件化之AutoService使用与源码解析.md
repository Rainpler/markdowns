AutoService是google提供的一款路由框架，组件化搭建的过程中需要模块间的通信功能，一般常用的是阿里的Arouter、CC等第三方库通信。相比于这两者，AutoService使用更方便，业务开发的过程中Bug更少，更加轻量化，实现了接口下沉，提供功能给组件用，且组件之间相互不依赖。首先先来看一下其简单的使用：

## 牛刀小试
我们通过一个简单的例子来使用一下AutoService，从而看一下它在使用上是多么的轻量。
#### 添加依赖
```java
dependencies{
   api 'com.google.auto.service:auto-service:1.0-rc4'
}
```
#### 编写接口
相关的autoservice接口我们在common层编写。
```java
public interface Processor {
  void process();
}
```

#### 接口实现
在我们的模块中，要去实现Processor接口，并且添加@AutoService的注释
```java
@AutoService(Processor.class)
public ProcessorImpl implements Processor{

  @Override
  void process(){

  }
}
```
#### 调用接口
调用接口实现，需要通过ServiceLoader来加载实现了`@AutoService(Processor.class)`注解的类。
```java
ServiceLoader.load(Processor.class).iterator().next().process();
```
ServiceLoader实现了Iterable接口，所以load()调用之后实际上得到是迭代器的对象。通过`.iterator().next()`方法可以拿到所有同一个Class接口的实现类。

为了方法的重用，也为了避免每次都写重复代码，因此将`load()`封装在base层作为基础方法供调用。
```java
public class IServiceLoader {
    public IServiceLoader() {
    }

    public static <T> T load(Class<T> service) {
        try {
            return ServiceLoader.load(service).iterator().next();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

## 庖丁解牛
AutoService实际上是APT技术的一个应用，APT全称为Annotation Processor Tools ，即注解处理器。通过注解处理器在编译期能够获取注解与注解声明的类包括类中所有成员信息。因为APT中注解只保留到源码级别，所以我们先来了解一下javac。

### Javac

Javac的编译过程，大致可以分为3个过程：
- 解析与填充符号表
- 插入式注解处理器的注解处理过程
- 分析与字节码生成过程

首先会进行词法和语法分析，词法分析将源代码的字符流转变为Token集合，关键字/变量名/字面量/运算符读可以成为Token，词法分析过程由com.sun.tools.javac.parserScanner类实现；

语法分析是根据Token序列构造抽象语法树的过程，抽象语法树AST是一种用来描述程序代码语法结构的树形表示，语法树的每一个节点读代表着程序代码中的一个语法结构，例如包/类型/修饰符/运算符/接口/返回值/代码注释等，在javac的源码中，语法分析是由com.sun.tools.javac.parser.Parser类实现，这个阶段产出的抽象语法树由com.sun.tools.javac.tree.JCTree类表示。经过上面两个步骤编译器就基本不会再对源码文件进行操作了，后续的操作读建立在抽象语法树上。

完成了语法和词法分析后就是填充符号表的过程。符号表是由一组符号地址和符号信息构成的表格。填充符号表的过程由com.sun.tools.javac.comp.Enter类实现。

如前面介绍的，如果注解处理器在处理注解期间对语法树进行了修改，编译器将回到解析与填充符号表的过程重新处理，直到所有插入式注解处理器都没有再对语法树进行修改为止，每一次循环称为一个Round。

### 实现原理
接下来看一下AutoService的实现原理。在其注释中，给定了以下三个限定条件：
- 不能是内部类和匿名类，必须要有确定的名称
- 必须要有公共的，可调用的无参构造函数
- 使用这个注解的类必须要实现value参数定义的接口

```java
@Documented
@Retention(CLASS)
@Target(TYPE)
public @interface AutoService {
  /** Returns the interface implemented by this service provider. */
  Class<?> value();
}
```

由于AutoService是在源码阶段的注解，我们知道，一个.java文件要由javac编译成.class文件，并交由虚拟机去运行。在这个过程中，javac会采集到所有的注解信息，对每一个每个语法树节点包装表示为一个Element，在javax.lang.model包中定义了16类Element，包括常用的元素：包，枚举，类，注解，接口，枚举值，字段，参数，本地变量，异常，方法，构造函数，静态语句块即static{}块，实例语句块即{}块，参数化类型即反省尖括号内的类型，还有未定义的其他语法树节点。


最后后交给注解处理程序处理。因此必须要有对应的注解处理器。

#### 注解处理器
自定义注解处理器，需要继承自`AbstractProcessor`，一般我们会实现其中的3个方法：
 - getSupportedAnnotationTypes() 返回了支持的注解类型
 - getSupportedSourceVersion()  用来指定支持的java版本，一般来说我们都是支持到最新版本，因此直接返回 SourceVersion.latestSupported()即可
 - process() 处理方法

#### AutoServiceProcessor
我们来看一下Google为AutoService实现的注解处理器AutoServiceProcessor。
```java
public class AutoServiceProcessor extends AbstractProcessor {

  @VisibleForTesting
  static final String MISSING_SERVICES_ERROR = "No service interfaces provided for element!";


  private Multimap<String, String> providers = HashMultimap.create();

  @Override
  public ImmutableSet<String> getSupportedAnnotationTypes() {
    return ImmutableSet.of(AutoService.class.getName());
  }

  @Override
  public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
  }


  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    try {
      return processImpl(annotations, roundEnv);
    } catch (Exception e) {
      // We don't allow exceptions of any kind to propagate to the compiler
      StringWriter writer = new StringWriter();
      e.printStackTrace(new PrintWriter(writer));
      fatalError(writer.toString());
      return true;
    }
  }
}
```
首先是三个基本方法，getSupportedAnnotationTypes() 返回了支持的注解类型AutoService.class。
getSupportedSourceVersion()返回了SourceVersion.latestSupported()，即支持到最新版本。而process()方法中又调用了processImpl()方法来进一步处理。
```java
private boolean processImpl(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
  if (roundEnv.processingOver()) {
    generateConfigFiles();
  } else {
    processAnnotations(annotations, roundEnv);
  }

  return true;
}
```
这里有两个逻辑，如果在上一次循环中已经注解处理器已经处理完了，那么就会调用`generateConfigFiles()`生成生成MEATA_INF配置文件，如果上一轮没有处理会调用`processAnnotations()`对注解进行处理。最后返回true表示已经处理完毕。
##### 环境变量
再接着往下看代码之前先看下两个环境变量，`RoundEnvironment`和`ProcessingEnvironment`。

RoundEnvironment提供了访问到当前这个Round中语法树节点的功能，errorRaised方法返回上一轮注解处理器是否产生错误；getRootElements返回上一轮注解处理器生成的根元素；最后两个方法返回包含指定注解类型的元素的集合，这个就是我们自定义注解处理器需要经常打交道的方法。

```java
public interface RoundEnvironment {
    boolean processingOver();

    boolean errorRaised();

    Set<? extends Element> getRootElements();

    Set<? extends Element> getElementsAnnotatedWith(TypeElement var1);

    Set<? extends Element> getElementsAnnotatedWith(Class<? extends Annotation> var1);
}
```

另外一个参数ProcessingEnvironment，定义在AbstractProcessor中，在注解处理器初始化的时候(init()方法执行的时候)创建。
```java
public synchronized void init(ProcessingEnvironment processingEnv) {
    if (this.initialized) {
        throw new IllegalStateException("Cannot call init more than once.");
    } else {
        Objects.requireNonNull(processingEnv, "Tool provided null ProcessingEnvironment");
        this.processingEnv = processingEnv;
        this.initialized = true;
    }
}
```
它代表了注解处理器框架提供的一个上下文环境，要创建新的代码或者向编译器输出信息或者获取其他工具类等都需要用到这个实例变量。看下它的源码：
```java
public interface ProcessingEnvironment {
    Map<String, String> getOptions();

    Messager getMessager();

    Filer getFiler();

    Elements getElementUtils();

    Types getTypeUtils();

    SourceVersion getSourceVersion();

    Locale getLocale();
}
```
- Messager用来报告错误，警告和其他提示信息;
- Filer用来创建新的源文件，class文件以及辅助文件;
- Elements中包含用于操作Element的工具方法;
- Types中包含用于操作类型TypeMirror的工具方法;

##### 处理注解
介绍完两个环境变量，接下来继续看`processAnnotations()`方法。
```java
private void processAnnotations(Set<? extends TypeElement> annotations,
      RoundEnvironment roundEnv) {

  Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(AutoService.class);

  log(annotations.toString());
  log(elements.toString());

  for (Element e : elements) {
    // TODO(gak): check for error trees?
    TypeElement providerImplementer = (TypeElement) e;
    AnnotationMirror annotationMirror = getAnnotationMirror(e, AutoService.class).get();
    Set<DeclaredType> providerInterfaces = getValueFieldOfClasses(annotationMirror);
    if (providerInterfaces.isEmpty()) {
        error(MISSING_SERVICES_ERROR, e, annotationMirror);
        continue;
    }
    for (DeclaredType providerInterface : providerInterfaces) {
      TypeElement providerType = MoreTypes.asTypeElement(providerInterface);

      log("provider interface: " + providerType.getQualifiedName());
      log("provider implementer: " + providerImplementer.getQualifiedName());

      if (checkImplementer(providerImplementer, providerType)) {
        providers.put(getBinaryName(providerType), getBinaryName(providerImplementer));
      } else {
        String message = "ServiceProviders must implement their service provider interface. "
            + providerImplementer.getQualifiedName() + " does not implement "
            + providerType.getQualifiedName();
        error(message, e, annotationMirror);
      }
    }
  }
}
```
该方法的结构很简单，首先通过RoundEnvironment的`getElementsAnnotatedWith()`方法拿到所有标注了AutoService注解的元素。然后通过for进行循环遍历，在遍历中将需要将得到的对象e强转为TypeElement对象providerImplementer，然后调用`MoreElements.getAnnotationMirror()`方法判断providerImplementer中是否有注解了AutoService.class，这里返回的是Optional对象，Optional 类的引入很好的解决空指针异常。
```java
 public static Optional<AnnotationMirror> getAnnotationMirror(Element element,
      Class<? extends Annotation> annotationClass) {
    String annotationClassName = annotationClass.getCanonicalName();
    for (AnnotationMirror annotationMirror : element.getAnnotationMirrors()) {
      TypeElement annotationTypeElement = asType(annotationMirror.getAnnotationType().asElement());
      if (annotationTypeElement.getQualifiedName().contentEquals(annotationClassName)) {
        return Optional.of(annotationMirror);
      }
    }
    return Optional.absent();
  }
```
拿到Optional<AnnotationMirror>类对象后，再通过`get()`方法拿到AnnotationMirror对象，通过它可以拿到注解类型和注解参数，然后调用this的`getValueFieldOfClasses()`方法拿到注解的参数value值，也就是接口名。
```java
private ImmutableSet<DeclaredType> getValueFieldOfClasses(AnnotationMirror annotationMirror) {
    return getAnnotationValue(annotationMirror, "value")
        .accept(
            new SimpleAnnotationValueVisitor8<ImmutableSet<DeclaredType>, Void>() {
              @Override
              public ImmutableSet<DeclaredType> visitType(TypeMirror typeMirror, Void v) {
                // TODO(ronshapiro): class literals may not always be declared types, i.e. int.class,
                // int[].class
                return ImmutableSet.of(MoreTypes.asDeclared(typeMirror));
              }

              @Override
              public ImmutableSet<DeclaredType> visitArray(
                  List<? extends AnnotationValue> values, Void v) {
                return values
                    .stream()
                    .flatMap(value -> value.accept(this, null).stream())
                    .collect(toImmutableSet());
              }
            },
            null);
}
```
在这里，由于AutoService注解的类有可能value值是一个列表，拿到的value值providerInterfaces其实是一个set集合。。接下来先做判空，如果不为空则进一步遍历providerInterfaces对象。

首先将得到的DeclaredType通过`MoreTypes.asTypeElement()`方法转型为TypeElement对象，这样可以进行类型擦除。

>你可以把DeclaredType （type）看作是一个类的泛型类型（例如`List<String>` ）; 与基本上忽略泛型类型的TypeElement （element）相比（例如`List` ）。

然后调用`checkImplementer()`对给定的class进行约束校验，检查类型T是不是实现了注解参数值说明的接口，并返回校验结果。
```java
private boolean checkImplementer(TypeElement providerImplementer, TypeElement providerType) {

  String verify = processingEnv.getOptions().get("verify");
  if (verify == null || !Boolean.valueOf(verify)) {
    return true;
  }

  // TODO: We're currently only enforcing the subtype relationship
  // constraint. It would be nice to enforce them all.

  Types types = processingEnv.getTypeUtils();

  return types.isSubtype(providerImplementer.asType(), providerType.asType());
}
```
如果检验通过，则会将providerType和providerImplementer两个对象的二进制名称保存到Map对象providers中，即将接口与实现类形成映射关系。以一开始使用AutoService的例子来看，得到的Map对象将会是Map<Processor ,ProcessorImpl>的形式。即以接口为key，实现类为value的Map。


到这里就扫描得到了AutoService标注的实现类和对应接口的映射关系，并且在processImpl里面返回了true。当下一次循环到来的时候，这时候就会走`generateConfigFiles()`方法了。

##### 生成配置文件
```java
private void generateConfigFiles() {
  Filer filer = processingEnv.getFiler();

   // 1.
  for (String providerInterface : providers.keySet()) {
    String resourceFile = "META-INF/services/" + providerInterface;
    log("Working on resource file: " + resourceFile);
    try {
      SortedSet<String> allServices = Sets.newTreeSet();
      try {
        // 2.
        FileObject existingFile = filer.getResource(StandardLocation.CLASS_OUTPUT, "",
            resourceFile);
        log("Looking for existing resource file at " + existingFile.toUri());
        // 3.
        Set<String> oldServices = ServicesFiles.readServiceFile(existingFile.openInputStream());
        log("Existing service entries: " + oldServices);
        allServices.addAll(oldServices);
      } catch (IOException e) {
        log("Resource file did not already exist.");
      }

      // 4.
      Set<String> newServices = new HashSet<String>(providers.get(providerInterface));
      if (allServices.containsAll(newServices)) {
        log("No new service entries being added.");
        return;
      }

      allServices.addAll(newServices);
      log("New service file contents: " + allServices);

      // 5.
      FileObject fileObject = filer.createResource(StandardLocation.CLASS_OUTPUT, "",
          resourceFile);
      OutputStream out = fileObject.openOutputStream();
      ServicesFiles.writeServiceFile(allServices, out);
      out.close();
      log("Wrote to: " + fileObject.toUri());
    } catch (IOException e) {
      fatalError("Unable to create " + resourceFile + ", " + e);
      return;
    }
  }
}
```
主要分成5个步骤要生成配置文件，分别来看下：
- 第一步遍历上面拿到的映射关系map providers，我们这里就是`com.king.demo.interfaces.Processor` ，然后生成文件名`META-INF/services/com.king.demo.interfaces.Processor`
- 第二步先检查下在类编译class输出位置有没有存在配置文件，
- 第三步就是如果第二步存在配置文件，需要把接口和所有实现类保存到allServices中
- 第四步检查`processAnnotations()`方法输出的映射map是否不存在上面的allServices，不存在则添加，存在则直接返回不需要生成新的文件
- 第五步就是通过Filer生成配置文件，文件名就是resourceFile，文件内容就是allServices中的所有实现类。

最后编译代码，就会在相应的module中生成配置文件。

当我们使用的时候，就可以使用过`ServiceLoader`通过反射拿到所有的实现类。关于ServiceLoader的原理会在下一篇文章进行分析。
