随着高级语言的横空出世，类似于java一样的基于面向对象的编程语言如今越来越多，尽管这类编程语言在语法风格上存在一定的差别，但是它们彼此之间始终保持着一个共性，那就是都支持封装，继承和多态等面向对象特性，Java中任何一个普通的方法其实都具备虚函数的特征，它们相当于C++语言中的虚函数，区别在于C++中需要使用关键字virtual来显式定义虚方法，如果在Java程序中不希望某个方法拥有虚函数的特征时，则可以使用关键字final来标记这个方法。

# 多态Polymorphism

虚方法具备多态特性，所谓多态就是指程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定，而是在程序运行期间才确定，即一个引用变量到底会指向哪个类的实例对象，该引用变量发出的方法调用到底是哪个类中实现的方法，编译期间无法确定，只能在程序运行期间决定。

1. 面向接口编程属于多态：

```Java
Base base = new Derive();
```

2. 此外利用Java反射实例化第三方定制的jar包用于系统灵活适配也属于多态。

```Java
Class clz = Class.forName(dbconfig.dbDriver);
try{
    Driver d = (Driver)clz.newInstance(dbconfig.dbUri, dbconfig.dbUsername, dbconfig.dbPassword);
}catch{
    // 如果dbDriver这个class找不到，又或者clz不是个Driver，抛异常就可以了
}
```

# 多态的底层原理

在JVM中，将符号引用转换为调用方法的直接引用与方法的绑定机制相关，绑定机制指的是一个字段、方法或者类在符号引用被替换为直接引用的过程，这个过程仅仅发生一次。JVM中支持两种绑定方式，分别是静态绑定和动态绑定。两者的区别在于，动态绑定可以最大程度地支持多态

- 静态绑定Early Binding，指非虚方法在编译期间就确定了具体的调用版本，此后在运行期保持不变，因此非虚方法不具备多态特性，比如静态方法、私有方法、final方法、实例构造器（构造器通过NEW的方式实例化对象，实例已经确定）、父类方法（super调用）。该场景称为静态编译。比如你写了`Foo f = new Foo()`编译器就知道这是个`Foo`
- 动态绑定Later Binding，如果被调用的方法在编译期无法被确定下来，比如`Base base = new Derive()`，编译器并不知道`base.func()`是哪个派生类的`func()`。只能够在程序运行期，JVM根据实际情况分析`base.class`字节码才能将调用方法的符号引用转换为直接引用，因此动态绑定也就被称为晚期绑定。动态绑定可以最大程度地支持多态，而多态最大的意义在于降低类的耦合性。

需要注意的是，在处理Java类中的成员变量时，并不是采用运行时绑定，而是一般意义上的静态绑定，因为属性不能被重写。下面设计一个向上转型案例来验证：
- 在父类和子类中同时定义和赋值同名的成员变量`name`，并在子类中试图输出该变量的值。
- 在子类中重写父类方法method()

```Java
 public class Father { 
   protected String name="父亲属性"; 
   
   public void method() { 
     System.out.println("父类方法，对象类型：" + this.getClass()); 
   } 
 } 
 
 public class Son extends Father { 
   protected String name="儿子属性"; 
   
   public void method() { 
     System.out.println("子类方法，对象类型：" + this.getClass()); 
   } 
   
   public static void main(String[] args) { 
     Father sample = new Son();//向上转型
     System.out.println("调用的方法："+sample.method()); 
     System.out.println("调用的成员："+sample.name); 
   } 
 } 
```

打印结果如下，所以即使在向上转型的情况下：
- 基类的句柄可以“找到”子类方法
- 但`基类的句柄.成员变量`依旧是父类的成员变量。

```Java
子类方法，对象类型：class samples.Son
调用的成员：父亲属性
```