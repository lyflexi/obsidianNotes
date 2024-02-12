>适配器模式与装饰者模式大量使用于Java的IO流

装饰器与适配器都有一个别名叫做 包装模式(Wrapper)，它们看似都是起到包装一个类或对象的作用，但是使用它们的目的很不一样。

**适配器模式**就是 你需要调用A，发现A不太符合，你需要用B类型进行适配，间接调用；这个B不继承A，没有继承关系，纯适配关系；体感：适配器使用起来就能看到和目标对象的类型不一样；**适配器和源角色之间没有继承关系**

**装饰器模式**就是 你原先调用的是A类型，现在调用还是A类型，只不过这个A类型是装饰器，只是看起来像A，因为继承A，有继承关系；**装饰器与原角色归属于同一接口，二者属于接口的不同实现，装饰器属于增强实现**

# 适配器模式

对适配器模式的功能很好理解，就是把一个类的接口变换成客户端所能接受的另一种接口。适配器模式的结构：

- target（目标要求）:所要转换的所期待的接口
    
- Adaptee（源角色）：需要适配的类
    
- Adapter（适配器）：将源角色适配成目标要求接口
    

在 JAVA的老版阻塞式BIO类库中有很多这样的需求。下面以`InputStreamReader`和`OutputStreamWriter` 类为例介绍适配器模式。`InputStreamReader` 和 `OutputStreamWriter` 分别继承`Reader`和`Writer`两个抽象类，但是要创建它们的对象必须在构造函数中传入一个 `InputStream`和 `OutputStream` 的实例。`InputStreamReader` 和 `OutputStreamWriter`也就是将 `InputStream`和 `OutputStream`适配到`Reader`和`Writer`。

```Java
//file 为已定义好的文件流 
FileInputStream fileInput = new FileInputStream(file); 
InputStreamReader inputStreamReader = new InputStreamReader(fileInput);
```

我们调`InputStream`发现不符合要求，客户要求调`Reader`，所以由`InputStreamReader`适配器的构造方法传入`InputStream`参数实现适配效果。

# 装饰者模式

动态地将责任附加到对象上，==若要扩展功能，装饰者模提供了比继承更有弹性的替代方案==，继承会破坏自身的单一职责原则（当然了，因为是面向接口的增强实现）。要求装饰对象和被装饰对象实现同一个接口，强调功能的增强

```Java
//file 为已定义好的文件流 
FileInputStream fileInput = new FileInputStream(file); 
//以下两行代码是装饰者模式
InputStreamReader inputStreamReader = new InputStreamReader(fileInput);
BufferedReader bufferedReader=new BufferedReader(inputStreamReader);
```

`InputStreamReader`源码：

```Java
public class InputStreamReader extends Reader {...}
```

`BufferedReader`源码：

```Java
public class BufferedReader extends Reader {...}
```

`Reader`源码：

```Java
public abstract class Reader implements Readable, Closeable {...}
```

会发现`InputStreamReader`和`BufferedReader`其实是面向接口编程的两种不同实现，`BufferedReader`是`InputStreamReader`的增强