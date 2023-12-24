字符串String有两种表示方式，分别是字面量和对象引用

```Java
String sl = "hello"；//字面量的定义方式；
String s2 = new String（"hello"）；//对象引用
```

JDK9的String源码示例：
- String类是已经被声明为`final`的， 不可被继承。
- String实现了`Serializable`接口：表示字符串是支持序列化的。
- String实现了`Comparable`接口：表示`String`可以比较大小
- String在jdk8及以前内部定义为`private final char value[]`用于存储字符串数据。自jdk9以来改成了`byte[]`节约了一些空间。因为Java中的char是2字节，byte是1字节

```Java
public final class String implements java.io.Serializable， Comparable<String>,CharSequence {
    @Stable
    private final byte[] value；
}
```

# String的不可变性体现

当对字符串进行加法赋值时，需要重新指定内存区域赋值，不能重复利用原有的内存空间。
因此一个简单的字符串加法操作要占用三个内存空间，空间利用率很低
![[Pasted image 20231224153854.png]]

# StringBuilder

因为`StringBuilder`的`append`方法效率要比`String`常量拼接高很多，所以目前`String`常量拼接本质上利用的是`StringBuilder.append`，最后再来一次`toString()`，但是在这个过程中会创建多个`StringBuilder`中间变量，内存占用更大；如果进行`GC`，需要花费额外的时间。
![[Pasted image 20231224160041.png]]
所以在实际开发中，建议显式使用`StringBuilder`，这样自始至终中只创建过一个`StringBuilder`的对象。
另外，如果基本确定要前前后后添加的字符串长度不高于某个限定值`highLevel`的情况下，建议手动指定`StringBuilder`容器上限大小。
```Java
StringBuilder s = new StringBuilder(highLevel);//new char[highLevel]
```
# 字符串常量池

为了使字符串在运行过程中速度更快、更节省内存，JVM给String提供了字符串常量池StringTable，字符串常量池StringTable位于JVM方法区

关于方法区的结构，在过去的版本jdk1.6/1.7/1.8当中均有变动，故在此提前声明：
- jdk1.6及之前：方法区被称为永久代（permanent generation） ，静态变量、字符串常量池存放在永久代上。
- jdk1.7：方法区仍然被称永久代，但已经“去永久代”，字符串常量池、静态变量移除，保存在堆中。
- jdk1.8及之后： 无永久代，方法区被称为元空间
    - 类型信息、字段、方法、常量保存在本地内存的元空间，
    - 字符串常量池、静态变量保留在堆空间
    - 元空间，不再使用虚拟机内存，而是使用本地内存。

![[Pasted image 20231224160112.png]]
![[Pasted image 20231224160119.png]]
![[Pasted image 20231224160127.png]]

## 字符串字面声明直接存入常量池1

```Java
@Test
public void test1() {
    String s1 = "abc";//字面量定义的方式，"abc"存储在字符串常量池中
    String s2 = "abc";//s1,s2指向同一个abc
    s1 = "hello";//s1指向字符串常量池新开辟了的hello，不影响原有的abc

    System.out.println(s1 == s2);//判断地址：true  --> false
    System.out.println(s1);//hello
    System.out.println(s2);//abc
}
```

`String str = "a"+ "b"` 会创建几个对象 ?
`javap -v StringNewTest.class` 反编译后， 部分片段如下，只有一个ldc指令
![[Pasted image 20231224160532.png]]

"a" + "b" 在编译时，就已经编译为 ab， 被放到常量池中。
等号左边的str只是作为引用，保存在栈中
所以（堆中）只有一个对象 ：ab
## 字符串构造声明也会存入常量池2

String构造方法本质也是字面量初始化，String构造出来的字符串也会存储在常量池中
![[Pasted image 20231224160542.png]]

`String str =new String(“ab”)`` 会创建几个对象？

```Java
public class StringNewTest {
    public static void main(String[] args) {
        String str = new String("ab");
    }
}
```

`javap -v StringNewTest.class` 反编译后， 部分片段如下，也出现了ldc指令
![[Pasted image 20231224160619.png]]
同样，等号左边的str只是作为引用保存在栈中，但是堆中会产生两个对象：
- 一个是堆中由构造产生的string
- 一个是字符串常量池中的string

## 显式调用toString()也会存入串常量池

实例对象显式调用toString()方法，会在字符串常量池当中生成相应字符串然后返回常量池当中的地址：
![[Pasted image 20231224160631.png]]

```Java
class Memory {
    private void foo(Object param) {
        String str = param.toString();
        System.out.println(str);
    }
    public static void main(String[] args) {
        int i = 1;
        Object obj = new Object();
        Memory mem = new Memory();
        mem.foo(obj);//line 5
    }
}
```

## 隐式调用toString()不会存入串常量池6

有别于上面toString()的显示调用，toString()的隐式调用不会出现字节码指令ldc，因此隐式调用toString()不会在字符串常量池当中生成相应字符串

```Java
new String("a") + new String("b")//共创建了几个对象
```

- 对象1【堆空间】： new String("a")-->对象2： 常量池中的"a"
- 对象3【堆空间】： new String("b")-->对象4： 常量池中的"b"
- 对象5【堆空间】：new StringBuilder()，变量拼接“+”操作肯定需要隐式提前创建了StringBuilder
- 对象6【堆空间】：toString()的隐式调用不会在字符串常量池当中生成相应字符串，拼接结果"ab"最终仅存在于堆区
![[Pasted image 20231224160703.png]]