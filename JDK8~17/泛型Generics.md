Java 泛型是JDK 5中引入的一个新特性，泛型提供了编译时类型安全检测机制，该机制允许在编译时检测到非法的类型。

泛型的本质是抽象化参数类型，也就是说所操作的数据类型被指定为一个参数，常用的通配符为： T，E，K，V，？
- ？ 表示不确定的 Java 类型
- T (type) 表示具体的一个 Java 类型
- K V (key value) 分别代表 Java 键值中的 Key Value
- E (element) 代表 Element

# 泛型应用场景

泛型有如下应用场景
- 可用于定义通用返回结果：`CommonResult<T>`通过参数 T 可根据具体的返回类型动态指定结果的数据类型
- 定义Excel处理类：`ExcelUtil<T>`用于动态指定 Excel 导出的数据类型
- 用于构建集合工具类。参考`Collections`中的`sort`、`binarySearch`方法
- ......

# 泛型擦除

Java 的泛型是伪泛型，这是因为在运行期间（反射期间），所有的泛型信息都会被擦掉，这也就是通常所说类型擦除

```Java
List<Integer> list = new ArrayList<>();
list.add(12);
list.add("a");//这里给整形容器添加字符型元素会报错
Class<? extends List> clazz = list.getClass();
Method add = clazz.getDeclaredMethod("add", Object.class);
//但是通过反射添加是可以的，这就说明在运行期间所有的泛型信息都会被擦掉
add.invoke(list, "kl");
System.out.println(list);
```

# 泛型解析

泛型擦除是有范围的，定义在类上的泛型信息是不会被擦除的。因为Java编译器在`class`类文件中以`Signature`属性的方式保留了类的泛型信息。

我们可以通过Type接口，在运行时获取泛型类`class`相关的信息，`Type`作为顶级接口，`Type`下还有几种类型比如
- `TypeVariable`
- `ParameterizedType`
- `WildCardType`
- `GenericArrayType`

```Java
// 抽象类，定义泛型<T>
public abstract class BaseDao<T> {
    public BaseDao(){
        //JVM分析class信息就是在运行时
        Class clazz = this.getClass();
        ParameterizedType  pt = (ParameterizedType) clazz.getGenericSuperclass(); 
        clazz = (Class) pt.getActualTypeArguments()[0];
        System.out.println(clazz);
    }
}

// 实现类
public class UserDao extends BaseDao<User> {
    public static void main(String[] args) {
    
        BaseDao<User> userDao = new UserDao();

    }
}
// 执行结果输出
class com.entity.User
```


除了上述常见的泛型类之外，Java泛型还支持泛型接口，泛型方法以及泛型参数：如下示例
泛型接口定义：
```Java
public interface Generator<T> {
    public T method();
}
```

实现泛型接口，不指定类型：

```Java
class GeneratorImpl<T> implements Generator<T>{
    @Override
    public T method() {
        return null;
    }
}
```

实现泛型接口，指定类型：

```Java
class GeneratorImpl implements Generator<String>{
    @Override
    public String method() {
        return "hello";
    }
}
```

泛型方法定义：含泛型参数与泛型返回值
```Java
public static <E> void printArray(E[] inputArray) {
    for (E element : inputArray) {
        System.out.printf("%s ", element);
    }
    System.out.println();
}
```

调用泛型方法：

```Java
// 创建不同类型数组： Integer和 Character
Integer[] intArray = { 1, 2, 3 };
String[] stringArray = { "Hello", "World" };
printArray(intArray);
printArray(stringArray);
```