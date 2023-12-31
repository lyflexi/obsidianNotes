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
```

上面简单介绍了一些Lambda表达式得好处与语法，我们知道使用Lambda表达式是需要使用函数式接口的，那么，岂不是在我们开发过程中需要定义许多函数式接口。其实不然，java8其实已经为我们定义好了四类内置函数式接口，这四类接口其实已经可以解决我们开发过程中绝大部分的问题。四大函数式接口指的是Consumer、Supplier、Function、Predicate，位于java.util.function包下
![[Pasted image 20240109200926.png]]
# `Consumer<T>`：消费型接口

消费型接口的内置方法`void accept(T t)`：有参数，无返回值类型

```Java
package java.util.function;

import java.util.Objects;

/**
 * Represents an operation that accepts a single input argument and returns no
 * result. Unlike most other functional interfaces, {@code Consumer} is expected
 * to operate via side-effects.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #accept(Object)}.
 *
 * @param <T> the type of the input to the operation
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * Returns a composed {@code Consumer} that performs, in sequence, this
     * operation followed by the {@code after} operation. If performing either
     * operation throws an exception, it is relayed to the caller of the
     * composed operation.  If performing this operation throws an exception,
     * the {@code after} operation will not be performed.
     *
     * @param after the operation to perform after this operation
     * @return a composed {@code Consumer} that performs in sequence this
     * operation followed by the {@code after} operation
     * @throws NullPointerException if {@code after} is null
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

应用示例

```Java
/**
     * 消费型接口Consumer<T>
     */
    @Test
    public void test1 () {
        //注意：System.out.println(x)并不代表Consumer接口的返回值，仅仅代表后续的业务逻辑处理
        consume(500, (x) -> System.out.println(x));
    }
    public void consume (double money, Consumer<Double> c) {
        c.accept(money);
    }
```

# `Supplier<T>`：供给型接口

供给型接口的内置方法`T get()`：只有返回值，没有入参

```Java
package java.util.function;

/**
 * Represents a supplier of results.
 *
 * <p>There is no requirement that a new or distinct result be returned each
 * time the supplier is invoked.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #get()}.
 *
 * @param <T> the type of results supplied by this supplier
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

应用示例：

```Java
/**
     * 供给型接口，Supplier<T>
     */
    @Test
    public void test2 () {
        Random ran = new Random();
        List<Integer> list = supplier(10, () -> ran.nextInt(10));
        for (Integer i : list) {
            System.out.println(i);
        }
    }
    /**
     * 随机产生sum个数量得集合
     * @param sum 集合内元素个数
     * @param sup
     * @return
     */
    public List<Integer> supplier(int sum, Supplier<Integer> sup){
        List<Integer> list = new ArrayList<Integer>();
        for (int i = 0; i < sum; i++) {
            list.add(sup.get());
        }
        return list;
    }
```

# `Function<T, R>`：函数型接口

函数型接口内置方法`R apply(T t)`：既要有返回值，也要有方法入参

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

应用示例：

```Java
/**
     * 函数型接口：Function<R, T>
     */
    @Test
    public void test3 () {
        String s = strOperar(" asdf ", x -> x.substring(0, 2));
        System.out.println(s);
        String s1 = strOperar(" asdf ", x -> x.trim());
        System.out.println(s1);
    }
    /**
     * 字符串操作
     * @param str 需要处理得字符串
     * @param fun Function接口
     * @return 处理之后得字符传
     */
    public String strOperar(String str, Function<String, String> fun) {
        return fun.apply(str);
    }
```

# `Predicate<T>`：断言型接口

断言型接口内置方法`boolean test(T t)`：输入一个参数，返回boolean值

```Java
package java.util.function;

import java.util.Objects;

/**
 * Represents a predicate (boolean-valued function) of one argument.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #test(Object)}.
 *
 * @param <T> the type of the input to the predicate
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Predicate<T> {

    /**
     * Evaluates this predicate on the given argument.
     *
     * @param t the input argument
     * @return {@code true} if the input argument matches the predicate,
     * otherwise {@code false}
     */
    boolean test(T t);

    /**
     * Returns a composed predicate that represents a short-circuiting logical
     * AND of this predicate and another.  When evaluating the composed
     * predicate, if this predicate is {@code false}, then the {@code other}
     * predicate is not evaluated.
     *
     * <p>Any exceptions thrown during evaluation of either predicate are relayed
     * to the caller; if evaluation of this predicate throws an exception, the
     * {@code other} predicate will not be evaluated.
     *
     * @param other a predicate that will be logically-ANDed with this
     *              predicate
     * @return a composed predicate that represents the short-circuiting logical
     * AND of this predicate and the {@code other} predicate
     * @throws NullPointerException if other is null
     */
    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    /**
     * Returns a predicate that represents the logical negation of this
     * predicate.
     *
     * @return a predicate that represents the logical negation of this
     * predicate
     */
    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    /**
     * Returns a composed predicate that represents a short-circuiting logical
     * OR of this predicate and another.  When evaluating the composed
     * predicate, if this predicate is {@code true}, then the {@code other}
     * predicate is not evaluated.
     *
     * <p>Any exceptions thrown during evaluation of either predicate are relayed
     * to the caller; if evaluation of this predicate throws an exception, the
     * {@code other} predicate will not be evaluated.
     *
     * @param other a predicate that will be logically-ORed with this
     *              predicate
     * @return a composed predicate that represents the short-circuiting logical
     * OR of this predicate and the {@code other} predicate
     * @throws NullPointerException if other is null
     */
    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    /**
     * Returns a predicate that tests if two arguments are equal according
     * to {@link Objects#equals(Object, Object)}.
     *
     * @param <T> the type of arguments to the predicate
     * @param targetRef the object reference with which to compare for equality,
     *               which may be {@code null}
     * @return a predicate that tests if two arguments are equal according
     * to {@link Objects#equals(Object, Object)}
     */
    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
```

应用示例：

```Java
/**
     * 断言型接口：Predicate<T>
     */
    @Test
    public void test4 () {
        List<Integer> l = new ArrayList<>();
        l.add(102);
        l.add(172);
        l.add(13);
        l.add(82);
        l.add(109);
        List<Integer> list = filterInt(l, x -> (x > 100));
        for (Integer integer : list) {
            System.out.println(integer);
        }
    }
    /**
     * 过滤集合
     * @param list
     * @param pre
     * @return
     */
    public List<Integer> filterInt(List<Integer> list, Predicate<Integer> pre){
        List<Integer> l = new ArrayList<>();
        for (Integer integer : list) {
            if (pre.test(integer))
                l.add(integer);
        }
        return l;
    }
```

# 函数式接口用于集合操作

List集合的超类Superinterfaces有`Collection`和`Iterable` ，其中`Iterable` 接口中存在forEach方法，forEach接收函数式接口参数`Consumer`
![[Pasted image 20240109200757.png]]

因此List可以通过forEach方法遍历出集合元素之后对每一个元素进行后续的业务逻辑处理

```Java
resources.forEach(item -> {});
```

其中`Collection`接口中存在`removeIf`方法，`removeIf`接收函数式接口参数`Predicate`
![[Pasted image 20240109200807.png]]
因此可以`removeIf`方法删除符合断言的元素

```Java
List<String> rancherClusterName = resClusterInfos.stream().map(ResClusterInfo::getPhysicalCluster).collect(Collectors.toList());

rancherClusterInfoList.removeIf(rancherClusterInfo -> {return rancherClusterName.contains(rancherClusterInfo.getClusterId());});
```

函数式接口还可用于遍历HashMap

```Java
Map<String,Object> map = new HashMap<>();
map.put("name","zhongxu");
map.put("age",28);
map.put("sex","man");
map.put("love","JAVA");
map.put("major","Java后端开发工程师");
//使用HashMap的foreach便利hashmap
System.out.println("使用lambda表达式遍历HashMap:");
map.forEach((key,value)->{ System.out.println(key+"-------"+value); });
//使用stream的foreach遍历hashmap
System.out.println("使用stream遍历HashMap:");
map.entrySet().stream().forEach(entry->{System.out.println(entry.getKey()+""+entry.getValue()); });
```
