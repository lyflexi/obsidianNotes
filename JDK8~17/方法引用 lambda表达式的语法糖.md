方法引用并不是传统的方法调用，方法引用是lambda表达式的语法糖
举例说明：下面是一个打印集合所有元素的例子，其中`value -> System.out.println(value)`是一个`Consumer`接口的lambda表达式实现，更进一步这个函数式接口可以通过方法引用来替换

```Java
public class TestArray {
    
    public static void main(String[] args) {
        List<String> list = Arrays.asList("xuxiaoxiao", "xudada", "xuzhongzhong");
        list.forEach(value -> System.out.println(value));
        //使用方法引用的方式，和上面的输出是一样的，方法引用使用的是双冒号`::`
        list.forEach(System.out::println);
    }
    /* 输出：
     * xuxiaoxiao
     * xudada
     * xuzhongzhong
     */
}
```

曾几何时，我对方法引用的理解很模糊，一般是使用IDE协助转换，但是转换后的写法很多我都不明白：
- Lambda的参数哪去了？
- 为什么::前面有的是类名称，有的是实例对象呢？
- 不是只有静态方法::前面才使用类名吗，怎么有的实例方法也使用类名啊？
- 为什么 ClassName::new 有时要求无参构造器，有时又要求有参构造器呢？构造器参数由什么决定呢？

解决纷繁复杂信息的最好方式就是分类，这里也不例外。方法引用可以分为分四类：
- ==调用类的静态方法==
- ==调用实例引用参数的方法==
- ==调用已经存在的实例引用的方法==
- ==调用类的构造函数==

在分类测试之前，我先定义一个测试类`TestUtil`，和一个`JavaBean(Student)`
- `TestUtil`里面有一个静态方法，一个实例方法
- `JavaBean(Student)`是一个普通实体类

具体如下代码所示
```java
public class MethodReference {
	...
  
  //示例类
  public static class TestUtil {
        public static boolean isBiggerThan3(int input) {
            return input > 3;
        }

        public void printDetail(Student student) {
            System.out.println(student.toString());
        }
    }

  public static class Student {
        private String name;
        private int age;

        public Student(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getStatus(String thing) {
            return String.format("%d岁的%s正在%s", age, name, thing);
        }

        @Override
        public String toString() {
            return "Student{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}
```
# 调用类的静态方法

lambda：有如下格式，args是参数，可以是多个，例如(a1,a2,a3)
```java
(args...) -> Class.staticMethod(args...)
```

method reference：不管有多少参数，都省略掉，编译器自动会帮我们传入
```java
Class::staticMethod
```

样例代码如下，其中isBiggerThan3 是TestUtil类的static方法。从下面的代码你可以清晰的看到，方法从匿名类到Lambda再到方法引用的演变。

```java
public void testStaticMethodRef() {
    //匿名内部类形式
    Predicate<Integer> p1 = new Predicate<Integer>() {
        @Override
        public boolean test(Integer integer) {
            return TestUtil.isBiggerThan3(integer);
        }
    };
    //lambda表达式形式
    Predicate<Integer> p2 = integer -> TestUtil.isBiggerThan3(integer);
    //MethodReference形式
    Predicate<Integer> p3 = TestUtil::isBiggerThan3;

    Stream.of(1, 2, 3, 4, 5).filter(p3).forEach(System.out::println);
}
```
# 调用传入的实例参数的方法
lambda：
```java
(obj, args...) -> obj.instanceMethod(args...)
```

method reference：假设`obj`为`ObjectType`的实例对象，这种方法引用看起来和静态方法那个一样。
```java
ObjectType::instanceMethod
```

样例代码如下：
```java
public void testInstanceMethodRef1() {
   //匿名内部类
    BiFunction<Student, String, String> f1 = new BiFunction<Student, String, String>() {
        @Override
        public String apply(Student student, String s) {
            return student.getStatus(s);
        }
    };
    
    //lambda
    BiFunction<Student, String, String> f2 = (student, s) -> student.getStatus(s);
    
	//method reference
    BiFunction<Student, String, String> f3 = Student::getStatus;

    System.out.println(getStudentStatus(new Student("erGouWang", 18), "study", f3));
}
private String getStudentStatus(Student student, String action, BiFunction<Student, String, String> biFunction) {
    return biFunction.apply(student, action);
}
```

我还记得过去参与过一个项目，项目代码如下：
```java
List<ResClusterInfo> resClusterInfos = resClusterInfoMapper.selectList(new EntityWrapper<>());
//lambda
List<String> rancherClusterName = resClusterInfos.stream()
	.map((resClusterInfo)->{resClusterInfo.getPhysicalCluster()})
    .collect(Collectors.toList());
//method reference
List<String> rancherClusterName = resClusterInfos.stream()
	.map(ResClusterInfo::getPhysicalCluster)
    .collect(Collectors.toList());
```
# 调用已经存在的实例的方法
lambda：
```java
(args...) -> obj.instanceMethod(args...)
```

method reference：我们观察一下上面的lambda表达式，发现obj对象不是当做参数传入的，而是已经存在的，所以写成方法引用时就是实例::方法
```java
obj::instanceMethod
```

代码样例：
```java
public void testInstanceMethodRef2() {
	//utilObj对象是我们提前new出来的，是已经存在了的对象，不是Lambda的入参。
    TestUtil utilObj = new TestUtil();
	//匿名内部类
    Consumer<Student> c1 = new Consumer<Student>() {
        @Override
        public void accept(Student student) {
            utilObj.printDetail(student);
        }
    };
    
    //Lambda表达式
    Consumer<Student> c2 = student -> utilObj.printDetail(student);
    
	//方法引用
    Consumer<Student> c3 = utilObj::printDetail;
    
	//使用
    consumeStudent(new Student("erGouWang", 18), c3);
}

private void consumeStudent(Student student, Consumer<Student> consumer) {
    consumer.accept(student);
}
```

# 调用类的构造函数
lambda：==值得注意的就是，ClassName类必须有一个与lambda表达式入参个数相匹配的构造函数。==
```java
(args...) -> new ClassName(args...)
```

method reference：
```java
ClassName::new
```

代码样例如下，Student类必须含有两个入参的构造函数，为什么呢？
```java
public void testConstructorMethodRef() {
	//匿名内部类
    BiFunction<String, Integer, Student> s1 = new BiFunction<String, Integer, Student>() {
        @Override
        public Student apply(String name, Integer age) {
            return new Student(name, age);
        }
    };
    
	//lambda表达式
    BiFunction<String, Integer, Student> s2 = (name, age) -> new Student(name, age);
    
	//对应的方法引用
    BiFunction<String, Integer, Student> s3 = Student::new;
    
	//使用
    System.out.println(getStudent("cuiHuaNiu", 20, s3).toString());
}

private Student getStudent(String name, int age, BiFunction<String, Integer, Student> biFunction) {
    return biFunction.apply(name, age);
}
```

因为我们的lambda表达式的类型是 BiFunction，而其正是通过两个入参来构建一个返回的。其签名如下：
通过入参`(t,u)`来生成`R`类型的一个值。
```java
@FunctionalInterface
public interface BiFunction<T, U, R> {
	  R apply(T t, U u);
	  ...
}
```

如果我们写成如下的方法引用，IDE就会报错，提示我们的Student类没有对应的构造函数
```java
Function<String, Student> s4 = Student::new;
```

这时，我们必须添加一个如下签名的构造函数才可以
```java
public Student(String name) {
    this.name = name;
}
```
# 留个作业🙂

熟悉了以上四种类型后，方法引用再也难不住你了！留个作业：
```java
Consumer<String> consumer1 = new Consumer<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
};

//lambda表达式
Consumer<String> consumer2 = ;

//方法引用
Consumer<String> consumer3 = ;
```

上面代码中的consumer2和consumer3分别是什么呢？其属于方法引用的哪一类呢？
答案如下：
```java
//lambda表达式  
Consumer<String> consumer2 = s -> System.out.println(s);  
//方式引用 println是System.out的静态方法，属于第一种情况  
Consumer<String> consumer3 = System.out::println;
```