与接口不同的是，抽象类中即可以包含抽象方法，也可以包含非抽象方法
- 若一个普通类实现了接口，那么，该普通类必须实现接口中所有的抽象方法。 
- 若一个抽象类实现了接口，那么，该抽象类可以实现接口中的抽象方法，可以实现部分，也可以不实现。
抽象类依然可以有抽象子类，若一个普通的子类继承了实现了某接口的抽象类：
- 该普通的子类只需额外实现抽象的父类尚未实现的接口里面的抽象方法，
- 也可以部分实现父类尚未实现的接口里面的抽象方法。
# 抽象类的使用场景：模板设计模式

考虑到接口可能有多个实现类，在实现类与接口之间增加一层抽象类能够减少代码冗余，复用共同的抽象类代码，定制化实现类的其余代码

拿个常用的例子说明，在异步网络请求中，经常使用类似以下接口：

```Java
public interface Callback<T> {
    void onSuccess(T data);
    void onFailure(Throwable e);
}
```

现在比如有一个界面，在该界面所有错误信息都直接提示信息，那就可以使用一个抽象类对所有错误做统一操作：

```Java
public abstract class RemoteCallback<T> implements Callback<T> {
    @Override
    public void onFailure(Throwable e) {
        getView().showMessage(e.getMessage());
    }
}
```

这样子就不用每个请求方法都实现onFailure(Throwable e)进行错误处理，只需要关心请求成功后都数据处理

```Java

//之后在这个界面上所有请求都可以使用类似下面都格式去完成：
request(new RemoteCallback<String>() {
    @Override
    public void onSuccess(String data) {

    }
});
```

如果某个错误请求比较特殊，覆盖重写onFailure(Throwable e)就可以单独错误处理。

现在如果换了个界面，在这个界面的所有错误信息都弹窗提示，那么这个界面的所有请求都直接使用另一个抽象类就可以了：

```Java
public abstract class RemoteCallback2<T> implements Callback<T> {
    @Override
    public void onFailure(Throwable e) {
        getView().showDialog(e.getMessage());
    }
}
```

## Spring设计

（Spring框架中大量使用了模板设计模式）

## JDK设计

JDK里有许多这样的设计，主要是做一个骨架实现，方便公用接口的一套实现，不必各自为营。

随便挑一个，看AbstractCollection开头的注释

```Java
public interface Collection<E> extends Iterable<E> 
public abstract class AbstractCollection<E> implements Collection<E>
```

看AbstractCollection开头的注释

```Java
This class provides a skeletal implementation of the Collection interface, 
to minimize the effort required to implement this interface. 
翻译一下也就是：
* 本类提供了对于接口Collection的一个骨架实现，它实现了部分的Collection接口。
* 开发者在需要实现Collection接口的时候可以直接继承AbstractCollection，而不需要再去实现一次Collection接口。
```