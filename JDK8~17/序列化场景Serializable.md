序列化：就是将对象转化成字节序列的过程。

反序列化：就是讲字节序列转化成对象的过程。

对象序列化成的字节序列会包含对象的类型信息、对象的数据等，说白了就是包含了描述这个对象的所有信息，能根据这些信息“复刻”出一个和原来一模一样的对象。

# 为什么要序列化？

那么为什么要去进行序列化呢？有以下两个原因

1. 持久化：对象是存储在JVM中的堆区的，但是如果JVM停止运行了，对象也不存在了。序列化可以将对象转化成字节序列，可以写进硬盘文件中实现持久化。在新开启的JVM中可以读取字节序列进行反序列化成对象。
2. 网络传输：网络直接传输数据，但是无法直接传输对象，可在传输前序列化，传输完成后反序列化成对象。所以所有可在网络上传输的对象都必须是可序列化的。

# JDK实现方式（两种）

怎么去实现对象的序列化呢？

Java为我们提供了对象序列化的机制：
1. 对于要序列化对象的类要去实现Serializable接口或者Externalizable接口
2. 实现方法：JDK提供的`ObjectOutputStream`和`ObjectInputStream`来实现序列化和反序列化

下面分别实现Serializable和Externalizable接口来演示序列化和反序列化

## 实现Serializable接口

```Java
public class TestBean implements Serializable {
    private Integer id;
    private String name;
    private Date date;
 //省去getter和setter方法和toString
}
```

序列化：

```Java
public static void main(String[] args) {
    TestBean testBean = new TestBean();
    testBean.setDate(new Date());
    testBean.setId(1);
    testBean.setName("zll1");
    //使用ObjectOutputStream序列化testBean对象并将其序列化成的字节序列写入test.txt文件
    try (FileOutputStream fileOutputStream = new FileOutputStream("D:\\test.txt");
         ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);) {
        objectOutputStream.writeObject(testBean);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

执行后可以在test.txt文件中看到序列化内容

反序列化：

```Java
public static void main(String[] args) {
    try (FileInputStream fileInputStream = new FileInputStream("D:\\test.txt");
         ObjectInputStream objectInputStream=new ObjectInputStream(fileInputStream)) {
        TestBean testBean = (TestBean) objectInputStream.readObject();
        System.out.println(testBean);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

输出结果

```Shell
TestBean{id=1, name='zll1', date=Fri Nov 27 14:52:48 CST 2020}
```

## 实现Externalizable接口

实现Externalizable接口必须重写两个方法

- writeExternal(ObjectOutput out)
    
- readExternal(ObjectInput in)
    

举个例子：

```Java
public class TextBean implements Externalizable {
    private Integer id;
    private String name;
    private Date date;

    //可以自定义决定那些需要序列化
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(id);
        out.writeObject(name);
        out.writeObject(date);
    }
    //可以自定义决定那些需要反序列化
    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        this.id = in.readInt();
        this.name = (String) in.readObject();
        this.date = (Date) in.readObject();
    }
    //省去getter和setter方法和toString
}
```

序列化：

```Java
public static void main(String[] args) {
    TextBean textBean = new TextBean();
    textBean.setDate(new Date());
    textBean.setId(1);
    textBean.setName("zll1");
    try (ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("D:\\externalizable.txt"))) {
        outputStream.writeObject(textBean);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

反序列化：

```Java
public static void main(String[] args) {
    try (ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("D:\\externalizable.txt"))) {
        TextBean textBean = (TextBean) objectInputStream.readObject();
        System.out.println(textBean);
        //输出结果：TextBean{id=1, name='zll1', date=Fri Nov 27 16:49:17 CST 2020}
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```
==注意事项：==
- Java序列化机制需要使用无参构造器创建对象实例，并通过对象的setter方法来设置实例变量的值。 如果一个对象没有无参构造器，那么在序列化的过程中就无法创建对象实例，从而导致序列化失败。 
- 一个对象要进行序列化，如果该对象成员变量是引用类型的，那这个引用类型的成员变量也一定要是可序列化的，否则序列化失败
- 对于不想序列化的字段可以再字段类型之前加上`transient`关键字修饰（反序列化时会被赋予默认值）
# serialVersionUID的作用

先讲述下序列化的过程：在进行序列化时，会把当前类的serialVersionUID写入到字节序列中，在反序列化时会将字节流中的serialVersionUID同本地对象中的serialVersionUID进行对比，一致的话进行反序列化，不一致则失败报错（报InvalidCastException异常），也就是说同一对象的serialVersionUID是唯一的：
- 同一个对象多次序列化成字节序列，这多个字节序列反序列化成的对象还是一个（使用`==`判断为true）（因为所有序列化保存的对象都会生成一个序列化编号，当再次序列化时回去检查此对象是否已经序列化了，如果是则不会重复序列化，而是返回上次序列化的编号）
- 如果序列化一个可变对象，修改序列化之后的对象属性值，再次序列化该对象只会保存上次序列化的编号（这是个坑注意下）

serialVersionUID的生成有三种方式（private static final long serialVersionUID= XXXL ）：

1. 显式声明：默认的1L
    
2. 显式声明：根据包名、类名、继承关系、非私有的方法和属性以及参数、返回值等诸多因素计算出的64位的hash值
    
3. 隐式声明：未显式的声明serialVersionUID时java序列化机制会根据Class自动生成一个serialVersionUID（最好不要这样，因为如果Class发生变化，自动生成的serialVersionUID可能会随之发生变化，导致匹配不上）
    

# Jackson库实现

其实对于对象转化成json字符串和json字符串转化成对象，也是属于序列化和反序列化的范畴，相对于JDK提供的序列化机制，各有各的优缺点：

- JDK序列化/反序列化：原生方法不依赖其他类库、但是不能跨平台使用。
    
- json序列化/反序列化：json字符串可读性高、可跨平台使用无语言限制、扩展性好。
    

如果你的项目进行了前后端分离，那你一定使用过JSON进行数据交互，那在后端就一定会涉及到对Json数据的解析，虽然使用SpringMvc加上@requestBody都已经帮我们解析好并映射到bean里了，但是他底层也是通过这些JSON解析类库来完成的==（SpringMVC默认使用的就是Jackson）==。在我们后端直接调其他服务的接口时，很多也会返回JSON数据也需要我们自己使用这些类库来进行解析。

常见的JSON解析类库：

- fastjson：阿里出品的一个json解析类库，很快，提供了很多静态方法使用方便，但是底层实现不是很好，解析过程中使用String的substring，性能很好，但是可能会导致内存泄漏。
    
- Gson：谷歌出品的json解析类库，但是性能相较于其他两个个稍微差点。
    
- Jackson：==相对比较推荐的一种JSON解析类库，性能好稳定。Jackson的源代码托管在：https://github.com/FasterXML/jackson。==
    

引入maven依赖：

```XML
<dependency>  
      <groupId>com.fasterxml.jackson.core</groupId>  
      <artifactId>jackson-databind</artifactId>  
      <version>${jackson-version}</version>  
</dependency>  
<dependency>  
      <groupId>com.fasterxml.jackson.core</groupId>  
      <artifactId>jackson-core</artifactId>  
      <version>${jackson-version}</version>  
</dependency>  
<dependency>  
      <groupId>com.fasterxml.jackson.core</groupId>  
      <artifactId>jackson-annotations</artifactId>  
      <version>${jackson-version}</version>  
</dependency> 
<!-- 其中Jackson Annotations依赖Jackson Core，Jackson Databind依赖Jackson Annotations。-->
```

## **序列化**

1. ObjectMapper（将JavaObject转化成JSON）

```Java
Element element = new Element();
element.setAge(1);
element.setElName("zll");
ObjectMapper objectMapper = new ObjectMapper();
String elementStr = objectMapper.writeValueAsString(element);
System.out.println(elementStr);
```

输出结果如下：

```Java
{"age":1,"elName":"zll"}
```

其他常用序列化方法：

- writeValue(File arg0, Object arg1)把arg1转成json序列，并保存到arg0文件中
    
- writeValue(OutputStream arg0, Object arg1)把arg1转成json序列，并保存到arg0输出流中。
    
- teValueAsBytes(Object arg0)把arg0转成json序列，并把结果输出成字节数组
    
- writeValueAsString(Object arg0)把arg0转成json序列，并把结果输出成字符串。

2. JsonGenerator（json生成器）：

可以根据自己的需要创建相应结构的json

```Java
        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        //JsonFactory jsonFactory = new JsonFactory();
        //创建jsonfactory 2种方法
        ObjectMapper objectMapper = new ObjectMapper();
        JsonFactory jsonFactory = objectMapper.getFactory();
        JsonGenerator generator = jsonFactory.createGenerator(bout);
        //创建自己需要的json
        //创建对象获取数组要写开始和结束
        generator.writeStartObject();
            //创建一个字段 第一个参数key 第二个参数value
            generator.writeStringField("name","value");
            generator.writeNumberField("numberName",1);
            //或者直接创建object
            generator.writeObjectField("ObjectName","ObjectValue");
            //创建数组
            generator.writeArrayFieldStart("arrayName");
                //里面可以是对象、数组、字符串、数字
                generator.writeString("element1");
                generator.writeNumber(1);
                generator.writeNumber(1);
            generator.writeEndArray();
        generator.writeEndObject();
        generator.flush();
        generator.close();
        String s = bout.toString();
        System.out.println(s);
```

执行结果：

```Java
{"name":"value","numberName":1,"ObjectName":"ObjectValue","arrayName":["element1",1,1]}
```

## **反序列化**

1. 使用ObjectMapper，将json字符串转成对象：
    

```Java
String str = "{\"id\":1,\"name\":\"haha\",\"elements\":[{\"age\":1,\"elName\":\"zll\"},{\"age\":2,\"elName\":\"zll1\"}]}";
ObjectMapper objectMapper = new ObjectMapper();
TestBean testBean = objectMapper.readValue(str, TestBean.class);
System.out.println(testBean.toString());
```

运行结果：

```Java
TestBean(id=1, name=haha, elements=[Element(age=1, elName=zll), Element(age=2, elName=zll1)])
```

2. 使用ObjectMapper，读取json某些字段值
    

```Java

    String str = "{\"id\":1,\"name\":\"haha\",\"elements\":[{\"age\":1,\"elName\":\"zll\"},{\"age\":2,\"elName\":\"zll1\"}]}";
    ObjectMapper objectMapper = new ObjectMapper();
    JsonNode jsonNode = objectMapper.readTree(str);
    //获取name字段值
    JsonNode name = jsonNode.get("name");
    String s = name.asText();
    System.out.println(s);
    //获取elements字段下数组第二个对象的age
    JsonNode elements = jsonNode.get("elements");
    JsonNode object2 = elements.get(1);//从0开始哦
    JsonNode age = object2.get("age");
    int i = age.asInt();
    System.out.println(i);
```

运行结果：

```Java
haha
2
```

## **ObjectMapper的常用设置**

```Java
ObjectMapper objectMapper = new ObjectMapper();  
 
//序列化的时候序列对象的所有属性  
objectMapper.setSerializationInclusion(Include.ALWAYS);  
 
//反序列化的时候如果多了其他属性,不抛出异常  
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);  
 
//如果是空对象的时候,不抛异常  
objectMapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);  
 
//属性为null的转换
objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
 
//取消时间的转化格式,默认是时间戳,可以取消,同时需要设置要表现的时间格式  
objectMapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);  
objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
```

## **常用注解**

1. @JsonIgnoreProperties(ignoreUnknown = true)：将这个注解加载类上，不存在的字段将被忽略。
    
2. @JsonIgnoreProperties({ “password”, “secretKey” })：指定忽略字段
    
3. @JsonIgnore：标在注解上，将忽略此字段
    
4. @JsonFormat(timezone = “GMT+8”, pattern = “yyyy-MM-dd HH:mm:ss”)：标在时间自端上序列化是使用制定规则格式化（默认转化成时间戳）
    
5. @JsonInclude(参数)JsonInclude.Include.NON_EMPTY：属性为空或者null都不参与序列化 JsonInclude.Include.NON_NULL：属性为null不参与序列化
    
6. @JsonProperty("firstName")标在字段上，指定序列化后的字段名
    
7. @JsonDeserialize(using= T extends JsonDeserializer.class)和@JsonSerialize(using= T extends JsonSerializer.class)，自定义某些类型字段的序列化与反序列化规则