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
