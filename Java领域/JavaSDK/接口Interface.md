Java语言规定接口不能被new，但是接口可以以匿名类内部类去new，之所以被称之为内部匿名类，是因为在编译阶段，编译器帮我们以内部类的形式，帮我们implement我们的接口，因此我们才可以以new的方式使用。

# 匿名类内部类

```Java
//Main 
public class Main {
    public static void main(String[] args) {
        final String name = "Haha";
        new FunLisenter() {
            @Override
            public void fun() {
                System.out.println(name);
            }
        }.fun();
    }
}

//FunLisenter 
public interface FunLisenter {
    void fun();
}
```

上述两个java文件编译后会生成三个class文件，分别是
- Main.class
- FunLisenter.class
- Main$1.class
使用`javap -c Main$1.class`命令反编译这个特别的Main$1.class文件

发现`Main$1.class`正是编译器对`FunLisenter`接口的具体实现

```Java
final class Main$1 implements FunLisenter {
    Main$1() {
    }

    public void fun() {
        //sleep(100000ms)
        System.out.println("Haha");
    }
}
```

# 外部变量必须声明为final

外部变量的生命周期与局部内部类的对象的生命周期的不一致性，假设`main()`方法被调用，从而在它的调用栈中生成了变量`name`，同时产生了一个局部内部类对象`FunLisenter`，`FunLisenter`访问了变量`name`，当`main()`方法运行结束后，局部变量`name`就已死亡了。但局部内部类对象`FunLisenter`还可能一直存在比如说它是一个睡眠了100s的线程，并没有随着`main()`方法的结束而立刻结束。这时就会出现一个荒唐的结果，局部内部类对象`FunLisenter`要访问一个已不存在的局部变量`name`

如何才能避免上述由生命周期不一致性而带来的荒唐结局呢？

当被声明的局部变量`name`是final类型时，final关键字使得局部变量`name`被复制了一份，复制品直接作为匿名内部类`FunLisenter`中的数据成员。这样当`FunLisenter`访问局部变量`name`时，其实真正访问的是这个局部变量的复制品，因此即使当`main`线程中真正的局部变量`name`死亡时，局部内部类`FunLisenter`仍可以访问外部`name`，给人的感觉好像是外部变量的生命期延长了。

因此就有了下面这段代码



```Java
public class Main {
    public static void main(String[] args) {
        final String name = "Haha";
        new FunLisenter() {
            @Override
            public void fun() {
                System.out.println(name);
            }
        }.fun();
    }
}
```

请注意局部变量`name`被声明为final之后，无论是修改外部的局部变量`name`，或是修改内部类的变量`name`都是被禁止的
- **虽然JDK8之后不需要final修饰了，但是如果代码中出现对变量重新赋值，是不能通过编译的。因为JDK8中默认变量是final类型，即所谓的effective final，因此还是要遵循final变量不能被修改的特性**
