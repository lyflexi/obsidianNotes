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
- `Main.class`
- `FunLisenter.class`
- `Main$1.class`
使用`javap -c Main$1.class`命令反编译这个特别的`Main$1.class`文件，发现`Main$1.class`正是编译器对`FunLisenter`接口的具体实现

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

## 1.外部变量必须声明为final

外部变量的生命周期与局部内部类的对象的生命周期的不一致性，假设`main()`方法被调用，从而在它的调用栈中生成了变量`name`，同时产生了一个局部内部类对象`FunLisenter`，`FunLisenter`访问了变量`name`，当`main()`方法运行结束后，局部变量`name`就已死亡了。但局部内部类对象`FunLisenter`还可能一直存在比如说它是一个睡眠了100s的线程，并没有随着`main()`方法的结束而立刻结束。这时就会出现一个荒唐的结果，局部内部类对象`FunLisenter`要访问一个已不存在的局部变量`name`

如何才能避免上述由生命周期不一致性而带来的荒唐结局呢？
当被声明的局部变量`name`是final类型时，final关键字使得局部变量`name`被复制了一份，复制品直接作为匿名内部类`FunLisenter`中的数据成员。这样当`FunLisenter`访问局部变量`name`时，其实真正访问的是这个局部变量的复制品，因此即使当`main`线程中真正的局部变量`name`死亡时，局部内部类`FunLisenter`仍可以访问外部`name`，给人的感觉好像是外部变量的生命期延长了。

因此就有了下面这段代码：
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
# 接口新特性
在JDK8的部分源码中，比如Java8的`Function`接口，我们可以发现以下几点不同：
- ==在接口中，如果接口当中只有一个传统的接口方法，那么后期实现接口的时候，匿名内部类就可以写成lambda表达式的形式==
- 在接口中，可以直接添加非抽象的实例方法{...}，用default修饰。此举可以删去很多冗余的abstract类，比如被弃用的WebMvcConfigurerAdapter抽象类（该抽象类从Spring5.0，也就是SpringBoot2.0后，已被弃用）
	1. 曾经的WebMvcConfigurerAdapter抽象类，实现了WebMvcConfigurer接口，但也只是放了一大堆空实现而已
	2. JDK8接口方法支持就地实现，只需要将方法声明为default即可，因此用户只需要实现即可WebMvcConfigurer
- 在接口中，可以直接添加静态方法。

## 1.支持函数式接口

函数式接口在JDK8中是指有且仅有一个抽象方法的接口。函数式接口可以通过Lambda表达式实现，Lambda表达式是匿名内部类的语法糖。

函数式接口中只能存在一个抽象方法，因为接口只有一个抽象方法的前提下，实例化接口的时候匿名内部类才唯一
比如
```Java
package java.util.function;

import java.util.Objects;

/**
 * Represents a function that accepts one argument and produces a result.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #apply(Object)}.
 *
 * @param <T> the type of the input to the function
 * @param <R> the type of the result of the function
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);

    /**
     * Returns a composed function that first applies the {@code before}
     * function to its input, and then applies this function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     *
     * @param <V> the type of input to the {@code before} function, and to the
     *           composed function
     * @param before the function to apply before this function is applied
     * @return a composed function that first applies the {@code before}
     * function and then applies this function
     * @throws NullPointerException if before is null
     *
     * @see #andThen(Function)
     */
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    /**
     * Returns a composed function that first applies this function to
     * its input, and then applies the {@code after} function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     *
     * @param <V> the type of output of the {@code after} function, and of the
     *           composed function
     * @param after the function to apply after this function is applied
     * @return a composed function that first applies this function and then
     * applies the {@code after} function
     * @throws NullPointerException if after is null
     *
     * @see #compose(Function)
     */
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    /**
     * Returns a function that always returns its input argument.
     *
     * @param <T> the type of the input and output objects to the function
     * @return a function that always returns its input argument
     */
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```
函数式接口使用举例：
用lambda表达式去实现`Runnable`接口，并将`Runnable`的实现传入`Thread`类的构造方法

```Java
public class ThreadBaseDemo
{
    public static void main(String[] args) throws InterruptedException
    {
        Thread t1 = new Thread(() -> {

        },"t1");
        t1.start();
    }
}

@FunctionalInterface  
public interface Runnable {  
    /**  
     * When an object implementing interface {@code Runnable} is used  
     * to create a thread, starting the thread causes the object's     * {@code run} method to be called in that separately executing  
     * thread.     * <p>  
     * The general contract of the method {@code run} is that it may  
     * take any action whatsoever.     *     * @see     java.lang.Thread#run()  
     */    public abstract void run();  
}
```
## 2.支持default实例方法

新接口中实例方法是接口自带的方法，实例方法的声明，需要增加default关键字修饰，因此这种方法也称为默认方法

接口被实现后，实例可以直接使用这些默认方法，同时如果对默认方法需要重写时，可以直接重写即可。

```Java
    /**
     * Returns a composed function that first applies the {@code before}
     * function to its input, and then applies this function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     *
     * @param <V> the type of input to the {@code before} function, and to the
     *           composed function
     * @param before the function to apply before this function is applied
     * @return a composed function that first applies the {@code before}
     * function and then applies this function
     * @throws NullPointerException if before is null
     *
     * @see #andThen(Function)
     */
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    /**
     * Returns a composed function that first applies this function to
     * its input, and then applies the {@code after} function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     *
     * @param <V> the type of output of the {@code after} function, and of the
     *           composed function
     * @param after the function to apply after this function is applied
     * @return a composed function that first applies this function and then
     * applies the {@code after} function
     * @throws NullPointerException if after is null
     *
     * @see #compose(Function)
     */
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }
```

## 3.支持接口静态方法

新接口中的静态方法作为接口的类方法，可以直接使用，不需要依托某个实现类。

```Java
    /**
     * Returns a function that always returns its input argument.
     *
     * @param <T> the type of the input and output objects to the function
     * @return a function that always returns its input argument
     */
    static <T> Function<T, T> identity() {
        return t -> t;
    }
```
