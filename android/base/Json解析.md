在之前的文章里面，我们讲解了关于序列化的知识，重点讲了Serializeble和Parcelable的使用，除此之外，还有Json、xml、protbuf等广义的序列化方案，但是Json是目前普及度最高的。

## Json基本知识
Json是什么？JSON（JavaScript Object NOtation）是一种轻量级的数据交换格式，作用是数据标记，存储，传输。在我们的日常开发中，就离不开Json，例如网络请求中的POST请求，我们通过是通过把数据包装成JSON进行传输。

#### 语法
JSON建构于两种结构：
- “名称/值”对的集合（A collection of name/value pairs）。不同的语言中，它被理解为对象（object），纪录（record），结构（struct），字典（dictionary），哈希表（hash table），有键列表（keyed list），或者关联数组 （associative array）。
- 值的有序列表（An ordered list of values）。在大部分语言中，它被理解为数组（array）。

这些都是常见的数据结构。事实上大部分现代计算机语言都以某种形式支持它们。这使得一种数据格式在同样基于这些结构的编程语言之间交换成为可能。

JSON具有以下这些形式：
对象是一个无序的“‘名称/值’对”集合。一个对象以“{”（左括号）开始，“}”（右括号）结束。每个“名称”后跟一个“:”（冒号）；“‘名称/值’ 对”之间使用“,”（逗号）分隔。
```json
{
  "name": "英语",
  "score": 78.3
}
```

数组是值（value）的有序集合。一个数组以“[”（左中括号）开始，“]”（右中括号）结束。值之间使用“,”（逗号）分隔。
```json
"courses": [
  {
    "name": "英语",
    "score": 78.3
  }
]
```

值（value）可以是双引号括起来的字符串（string）、数值(number)、true 、false 、 null 、对象（object）或者数组（array）。这些结构可以嵌套。

```json
{
  "url": "https://qqe2.com",
  "name": "欢迎使用JSON在线解析编辑器",
  "array": {
  "JSON校验": "http://jsonlint.qqe2.com/",
  "Cron生成": "http://cron.qqe2.com/",
  "JS加密解密": "http://edit.qqe2.com/"
  },
  "boolean": true,
  "null": null,
  "number": 123,
  "object": {
    "a": "b",
    "c": "d",
    "e": "f"
  }
}
```
字符串（string）是由双引号包围的任意数量Unicode字符的集合，使用反斜线转义。一个字符（character）即一个单独的字符串（character string）。字符串（string）与C或者Java的字符串非常相似。数值（number）也与C或者Java的数值非常相似。除去未曾使用的八进制与十六进制格式。

数值（number）也与C或者Java的数值非常相似。除去未曾使用的八进制与十六进制格式。
## Json解析
#### Android Studio自带org.json解析
解析原理：基于文档驱动，需要把全部文件读入到内存中，然后遍历所有数据，根据需要检索想要的数据。

具体使用请看下面的例子:
**生成JSON**
```java
private JSONObject createJson(Context context) throws Exception {
      File file = new File(getFilesDir(), "orgjson.json");//获取到应用在内部的私有文件夹下对应的orgjson.json文件

      JSONObject student = new JSONObject();//实例化一个JSONObject对象
      student.put("name", "OrgJson");//对其添加一个数据
      student.put("sax", "男");
      student.put("age", 23);

      JSONObject course1 = new JSONObject();
      course1.put("name", "语文");
      course1.put("score", 98.2f);

      JSONObject course2 = new JSONObject();
      course2.put("name", "数学");
      course2.put("score", 93.2f);

      JSONArray coures = new JSONArray();//实例化一个JSON数组
      coures.put(0, course1);//将course1添加到JSONArray，下标为0
      coures.put(1, course2);
      //然后将JSONArray添加到名为student的JSONObject
      student.put("courses", coures);

      FileOutputStream fos = new FileOutputStream(file);
      fos.write(student.toString().getBytes());
      fos.close();
      Log.i(TAG, "createJson: " + student.toString());
      Toast.makeText(context, "创建成功", Toast.LENGTH_LONG).show();
}
```
**解析JSON**
```java
private void parseJson(Context context) throws Exception {
      File file = new File(getFilesDir(), "orgjson.json");
      FileInputStream fis = new FileInputStream(file);
      InputStreamReader isr = new InputStreamReader(fis);
      BufferedReader br = new BufferedReader(isr);
      String line;
      StringBuffer sb = new StringBuffer();
      while (null != (line = br.readLine())) {
          sb.append(line);
      }
      fis.close();
      isr.close();
      br.close();
      Student student = new Student();
      //利用JSONObject进行解析
      JSONObject stuJsonObject = new JSONObject(sb.toString());
      //为什么不用getString?
      //optString会在得不到你想要的值时候返回空字符串""，而getString会抛出异常
      String name = stuJsonObject.optString("name", "");
      student.setName(name);
      student.setSax(stuJsonObject.optString("sax", "男"));
      student.setAge(stuJsonObject.optInt("age", 18));
      //获取数组数据
      JSONArray couresJson = stuJsonObject.optJSONArray("courses");
      for (int i = 0; i < couresJson.length(); i++) {
          JSONObject courseJsonObject = couresJson.getJSONObject(i);
          Course course = new Course();
          course.setName(courseJsonObject.optString("name", ""));
          course.setScore((float) courseJsonObject.optDouble("score", 0));
          student.addCourse(course);
      }
      Log.i(TAG, "parseJson: " + student);
      Toast.makeText(context, "解析成功", Toast.LENGTH_LONG).show();
}
```

#### Jackson解析
- 解析原理：基于事件驱动
- 解析过程：
  1. 类似 GSON，先创建1个对应于JSON数据的JavaBean类，再通过简单操作即可解析
  2. 与 Gson解析不同的是：GSON可按需解析，即创建的JavaBean类不一定完全涵盖所要解析的JSON数据，按需创建属性；但Jackson解析对应的JavaBean必须把Json数据里面的所有key都有所对应，即必须把JSON内的数据所有解析出来，无法按需解析.



依赖：
```java
implementation 'com.fasterxml.jackson.core:jackson-databind:2.9.8'
implementation 'com.fasterxml.jackson.core:jackson-core:2.9.8'
implementation 'com.fasterxml.jackson.core:jackson-annotations:2.9.8'
```
使用：
```java
public static void main(String... args) throws Exception {
    Student student = new Student();
    student.setName("杰克逊");
    student.setSax("男");
    student.setAge(28);
    student.addCourse(new Course("英语", 78.3f));
    student.addCourse(new Course("语文", 88.9f));
    student.addCourse(new Course("数学", 48.2f));

    ObjectMapper objectMapper = new ObjectMapper();
    //jackson序列化
    File file = new File(CurPath + "/jacksontest.json");
    FileOutputStream fileOutputStream = new FileOutputStream(file);
    objectMapper.writeValue(fileOutputStream, student);
    //反序列化
    Student student1 = objectMapper.readValue(file, Student.class);
    System.out.println(student1);
}
```
#### Gson解析字符串
- 解析原理：基于事件驱动
- 解析流程：根据所需取的数据 建立1个对应于JSON数据的JavaBean类，即可通过简单操作解析出
- 所需数据：Gson 不要求JavaBean类里面的属性一定全部和JSON数据里的所有key相同，可以按需取数据

具体实现：
1. 创建一个与JSON数据对应的JavaBean类（用作存储需要解析的数据）
    - JSON的大括号对应一个对象
    - JSON的方括号对应一个数组
    - 对象里可以有值/对象
    - 若对象里面只有值，没有key,则说明是纯数组，对应JavaBean里的数组类型
    - 若对象里面有值和key,则说明是对象数组，对应JavaBean里的内部类
    - 对象嵌套
      - 建立内部类，该内部类对象的名字父对象的key ，类似对象数组

依赖：
```java
implementation 'com.google.code.gson:gson:2.8.5'
```

Json例子：
```json
{
    "key": "value",
    "simpleArray": [1,2,3],
    "arrays": [
      [{
        "arrInnerClsKey": "arrInnerClsValue",
        "arrInnerClsKeyNub": 1
        }]
    ],
    "innerclass": {
      "name": "zero",
      "age": 25,
      "sax": "男"
    }
}
```
转化为JavaBean：
```java

public class GsonBean {
    private String key;
    private List<Integer> simpleArray;
    private List<List<ArraysBean>> arrays;
    private InnerclassBean innerclass;

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }

    public InnerclassBean getInnerclass() {
        return innerclass;
    }

    public void setInnerclass(InnerclassBean innerclass) {
        this.innerclass = innerclass;
    }

    public List<Integer> getSimpleArray() {
        return simpleArray;
    }

    public void setSimpleArray(List<Integer> simpleArray) {
        this.simpleArray = simpleArray;
    }


    public List<List<ArraysBean>> getArrays() {
        return arrays;
    }

    public void setArrays(List<List<ArraysBean>> arrays) {
        this.arrays = arrays;
    }

    public static class InnerclassBean {
        private String name;
        private int age;
        private String sax;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        public String getSax() {
            return sax;
        }

        public void setSax(String sax) {
            this.sax = sax;
        }
    }

    public static class ArraysBean {
        private String arrInnerClsKey;
        private int arrInnerClsKeyNub;

        public String getArrInnerClsKey() {
            return arrInnerClsKey;
        }

        public void setArrInnerClsKey(String arrInnerClsKey) {
            this.arrInnerClsKey = arrInnerClsKey;
        }

        public int getArrInnerClsKeyNub() {
            return arrInnerClsKeyNub;
        }

        public void setArrInnerClsKeyNub(int arrInnerClsKeyNub) {
            this.arrInnerClsKeyNub = arrInnerClsKeyNub;
        }
    }
}
```

使用：
```java
public static void main(String... args) throws Exception {
    Student student = new Student();
    student.setName("Zero");
    student.setSax("男");
    student.setAge(28);
    student.addCourse(new Course("英语", 78.3f));
    Gson gson = new Gson();
    //1. 生成json文件
    File file = new File(CurPath + "/gsonjsontest.json");
    OutputStream oot = new FileOutputStream(file);
    JsonWriter jw = new JsonWriter(new OutputStreamWriter(oot, "utf-8"));
    gson.toJson(student, new TypeToken<Student>() {}.getType(), jw);
    jw.flush();
    jw.close();
    //2. 反序列化
    Student student1 = gson.fromJson(new JsonReader(
        new InputStreamReader(new FileInputStream(file))),
        new TypeToken<Student>() {}.getType());
    System.out.println(student1);
}
```
##### Gson注解的使用
**@SerializedName** 注解可以将属性重命名，可以将json中的属性名转为我们自己自定义的属性名。假如后台传过来的字段为：
```json
"Data": {
		"1": 1000000001,
		"2": 2800,
		"3": "UPDATE",
		"4": 201901021130222,
	}
```
我们不可能创建这么一个Bean类去解析
```java
public class DataBean{
  private int 1;
  private int 2;
  private String 3;
  private long 4;
}
```
那这一定是非法的。而``@SerializedName``为我们提供了一个解决方案。先来看它的定义：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface SerializedName {

  /**
   * @return the desired name of the field when it is serialized or deserialized
   */
  String value();
  /**
   * @return the alternative names of the field when it is deserialized
   */
  String[] alternate() default {};
}
```
``@SerializedName``注解提供了两个属性，'value`可用于映射JSON中的属性，还有一个属性'alternate'：接收一个String数组。
alternate数组中出现任意一个属性名都可以转换为自定义的属性，如果出现多个则以最后一个为准。所以，我们的Bean类可以这么定义。

所以，可以使用注解 @SerializedName 方式来保证命名规范，同时又可以正常映射接口字段，如果接口字段和命名规则差别很大，使用@SerializedName 注解来解决还是有必要的。
```java
public class DataBean{
  @SerializedName("1")
  private int a;
  @SerializedName("2")
  private int b;
  @SerializedName("3")
  private String c;
  @SerializedName("4")
  private long d;
}
```

**@Expose** 注解可用serialize指定某字段是否参与序列化，默认为true，用deserialize指定是否参与反序列化，默认为true。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Expose {

  public boolean serialize() default true;
  public boolean deserialize() default true;
}
```
Gson在做版本控制的时候为我们提供了``@Since``和``@Until``两个注解，当我们做版本升级的时候，添加或是删改了某些字段，可以做兼容处理。

**@Since** 注解需传入一个double型变量n，表示当前版本≥n时，才会参与序列化/反序列化。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.TYPE})
public @interface Since {
  double value();
}
```
**@Until** 注解需传入一个double型变量n，表示当前版本≤n时，才会参与序列化/反序列化。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.TYPE})
public @interface Until {
  double value();
}
```

**@JsonAdapter** 平时我们去解析一个JSON的时候，一般都是通过反射去得到一个实体对象的，因此性能也较低。因此，我们可以通过JsonAdapter，去手动解析。

自定义adapter需要继承自``TypeAdapter``并实现Bean类的泛型。在``write()``方法中把Bean类转化成Json字符串，在``read()``方法中，解析Json字符串创建成Bean类。
```java
class UserJsonAdapter extends TypeAdapter<User> {
    @Override
    public void write(JsonWriter out, User user) throws IOException {
        out.beginObject();
        out.name("name");
        out.value(user.firstName + " " + user.lastName);
        out.endObject();
    }

    @Override
    public User read(JsonReader in) throws IOException {
        in.beginObject();
        in.nextName();
        String[] nameParts = in.nextString().split(" ");
        in.endObject();
        return new User(nameParts[0], nameParts[1]);
    }
}
```
我们来看这样一个串Json：
```json
{
   "name": "java",
   "authors": ""
}
```
后台返回的Json中有可能有空的String，这时候客户端使用默认的Gson解析就可能会报错，如果后台能重新发版还好，如果不能的话，那么客户端是不是可以在后台传值有问题的情况下，也进行错误处理呢。利用``TypeAdapter``，我们可以在``write()``中进行非空判断。
```java
new TypeAdapter<Foo>() {

        @Override
        public void write(JsonWriter out, Foo value) throws IOException {
            if (value == null) {//进行非空判断
                out.nullValue();
                return;
            }
        }

        @Override
        public Foo read(JsonReader in) throws IOException {
            if (in.peek() == JsonToken.NULL) {//进行非空判断
                in.nextNull();
                return null;
            }
        }
    }
```
或者使用自定义``JsonDeserializer``
```java
static class GsonError1Deserializer implements JsonDeserializer<GsonError> {

        @Override
        public GsonError deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context) throws JsonParseException {
            final JsonObject jsonObject = json.getAsJsonObject();

            final JsonElement jsonTitle = jsonObject.get("name");
            final String name = jsonTitle.getAsString();

            JsonElement jsonAuthors = jsonObject.get("authors");

            GsonError gsonError = new GsonError();

            if (jsonAuthors.isJsonArray()) {//如果数组类型，此种情况是我们需要的
                //关于context在文章最后有简单说明
                GsonError.AuthorsBean[] authors = context.deserialize(jsonAuthors, GsonError.AuthorsBean[].class);
                gsonError.setAuthors(Arrays.asList(authors));
            } else {//此种情况为无效情况
                gsonError.setAuthors(null);
            }
            gsonError.setName(name);
            return gsonError;
        }
    }
```
##### Gson 原理
>Gson在这个序列化和反序列化的过程中，充当的了一个解析器的角色。

**JsonElement**
该类是一个抽象类，代表着json串的某一个元素。这个元素可以是一个Json(JsonObject)、可以是一个数组(JsonArray)、可以是一个Java的基本类型(JsonPrimitive)、当然也可以为null(JsonNull);
![](http://darryrzhong.xyz/2019/09/15/java%E7%B3%BB%E5%88%97%E4%B9%8Bjson%E8%A7%A3%E6%9E%90/1240-20200309133433270.png)
JsonObject,JsonArray,JsonPrimitive,JsonNull都是JsonElement这个抽象类的子类。JsonElement提供了一系列的方法来判断当前的JsonElement。JsonObject对象可以看成 name/values的集合，而这些values就是一个个JsonElement。

**适配器模式**
当输入一个Json字符串的时候，首先通过``Type``判断是否是基本数据类型，如果是则创建对应的TypeAdapter读取json串并返回，如果不是，则创建ReflectTypeAdapter，然后遍历对象的属性。这就是Json解析的基本流程。像上面代码，我们O在Android中的应用O在Android中的应用O在Android中的应用也可以使用自定义TypeAdapter。

**Gson的整体解析流程**
![](http://darryrzhong.xyz/2019/09/15/java%E7%B3%BB%E5%88%97%E4%B9%8Bjson%E8%A7%A3%E6%9E%90/1240-20200309133444236.png)

![](http://darryrzhong.xyz/2019/09/15/java%E7%B3%BB%E5%88%97%E4%B9%8Bjson%E8%A7%A3%E6%9E%90/1240-20200309133448072.png)

**Gson的反射解析机制**

![](http://darryrzhong.xyz/2019/09/15/java%E7%B3%BB%E5%88%97%E4%B9%8Bjson%E8%A7%A3%E6%9E%90/1240-20200309133452928.png)
