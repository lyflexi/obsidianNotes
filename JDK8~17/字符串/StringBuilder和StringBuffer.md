String字符串常量的值是不可变的，这就导致每次对String的操作都会生成新的String对象，这样不仅效率低下，而且大量浪费有限的内存空间。而且StringBuffer与StringBuilder是字符串变量，StringBuffer和StringBuilder能够被多次的修改，并且不产生新的未使用对象
- String：操作少量的数据，空间利用率差，不要使用String类的"+"来进行频繁的拼接，因为那样的性能极差
- StringBuffer：单线程操作大量数据，效率低、线程安全
- StringBuilder：多线程操作大量数据，效率高、线程不安全

# StringBuffer

字符串变量（Synchronized，即线程安全）
如果要频繁对字符串内容进行修改，考虑最好使用StringBuffer，如果想转成String类型，可以调用StringBuffer的toString()方法。

StringBuffer 上的主要操作是 `append` 和 `insert` 方法，每个方法都能有效地将给定的数据转换成字符串，然后将该字符串的字符追加或插入到字符串缓冲区中。

- `append` 方法始终将这些字符添加到缓冲区的末端
- `insert` 方法则在指定的点添加字符。例如，如果 x 引用一个当前内容是`"start"`的字符串缓冲区对象，则此方法调用 `x.append("le")` 会使字符串缓冲区包含`"startle"`，而 `x.insert(4, "le")` 的结果是`"starlet"`

# StringBuilder

字符串变量（非线程安全）。在内部，StringBuilder对象被当作是一个包含字符序列的变长数组。其构造方法如下：

```Java
StringBuilder() //创建一个容量为16的StringBuilder对象（16个空元素）
StringBuilder(int initCapacity) //创建一个容量为initCapacity的StringBuilder对象
StringBuilder(String s) //创建一个包含s的StringBuilder对象，末尾附加16个空元素
```

在大部分情况下，StringBuilder的效率高于StringBuffer