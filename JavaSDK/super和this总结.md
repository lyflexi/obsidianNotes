# this

this 是自身的一个对象，代表对象本身，可以理解为：**指向对象本身的一个指针**。

this 的用法在 Java 中大体可以分为3种：
1. 指向对象本身的一个指针，这种就不用讲了，this 相当于是指向当前对象本身。
2. 调用当前类的成员变量或者成员方法：
	- 形参与成员名字重名，用 this 来区分。
	- 调用当前类的成员方法this.func()，我们总是会省略~~this~~.func()
	```java
class Person {
    private int age = 10;
    public Person(){
    System.out.println("初始化年龄："+age);
	}
    public int GetAge(int age){
	    //可以看到，这里 age 是 GetAge 成员方法的形参，this.age 是 Person 类的成员变量。
        this.age = age;
        return this.age;
    }
}
 
public class test1 {
    public static void main(String[] args) {
        Person Harry = new Person();
        System.out.println("Harry's age is "+Harry.GetAge(12));
    }
}
```

运行结果：
```java
初始化年龄：10
Harry's age is 12
```

3.引用构造函数

这个和 super 放在一起讲，见下面。

# super

super 可以理解为是指向自己超（父）类对象的一个指针，而这个超类指的是离自己最近的一个父类。

super 也有三种用法：

1. 普通的直接引用，与 this 类似，super 相当于是指向当前对象的父类的指针
2. 调用父类中的成员变量或者成员方法，这样就可以用 super.xxx 来引用父类的成员变量或者成员函数，如下代码实例：
```java
class Country {
    String name;
    void value() {
       name = "China";
    }
}
  
class City extends Country {
    String name;
    void value() {
	    name = "Shanghai";
	    System.out.println(name);
	    super.value();      //调用父类的方法
	    System.out.println(super.name);//调用父类的成员变量
	}
  
    public static void main(String[] args) {
	       City c=new City();
	       c.value();
       }
}
```

运行结果如下，
```java
Shanghai
China
```



3. 引用构造函数对比
	- super(参数)：调用父类中的某个构造函数（应为当前构造函数中的第一条语句）。==如果在子类构造函数位置调用super(参数)，则可以省略~~super(参数)~~，因为子类构造一定会调用父类构造==
	- this(参数)：调用本类中其他形式的构造函数（应为当前构造函数中的第一条语句）

```java
package com.javadebug.test;  
  
public class TestSuper {  
    public static class Person {  
  
        Person() {  
            System.out.println("父类·无参数构造方法： "+"A Person.");  
        }//构造方法(1)  
  
        Person(String name) {  
            System.out.println("父类·含一个参数的构造方法： "+"A person's name is " + name);  
        }//构造方法(2)  
    }  
  
    public static class Chinese extends Person {  
        Chinese() {  
//            super(); // 调用父类构造方法（1）  
            System.out.println("子类·调用父类无参数构造方法： "+"A chinese coder.");  
        }  
  
        Chinese(String name) {  
//            super(name);// 调用父类具有相同形参的构造方法（2）  
            System.out.println("子类·调用父类含一个参数的构造方法： "+"his name is " + name);  
        }  
  
        Chinese(String name, int age) {  
            this(name);// 调用具有相同形参的构造方法（3）  
            System.out.println("子类：调用子类具有相同形参的构造方法：his age is " + age);  
        }  
    }  
    public static void main(String[] args) {  
        Chinese cn = new Chinese();  
        cn = new Chinese("codersai");  
        cn = new Chinese("codersai", 18);  
    }  
}
```

例子中 Chinese 类第三种构造方法调用的是本类中第二种构造方法，而第二种构造方法是调用父类的，因此也要先调用父类的构造方法，再调用本类中第二种，最后是重写第三种构造方法。
```java
父类·无参数构造方法： A Person.
子类·调用父类”无参数构造方法“： A chinese coder.
父类·含一个参数的构造方法： A person's name is codersai
子类·调用父类”含一个参数的构造方法“： his name is codersai
父类·含一个参数的构造方法： A person's name is codersai
子类·调用父类”含一个参数的构造方法“： his name is codersai
子类：调用子类具有相同形参的构造方法：his age is 18
```


