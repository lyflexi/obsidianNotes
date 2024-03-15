`<init>` 和 `<clinit>` 是Java字节码中的两个特殊方法。
1. `<init>` 方法是构造函数的表示，用于对象的初始化。每个类可以有一个或多个构造函数，它们负责在创建对象时进行初始化工作。在Java字节码中，构造函数被命名为 `<init>`，并且在字节码中对应着实例初始化方法。
2. `<clinit>` 方法是类初始化方法，用于执行类的初始化操作。它包括静态变量的初始化和静态代码块的执行。在Java字节码中，类初始化方法被命名为 `<clinit>`，并且在字节码中对应着类初始化方法。
测试程序如下：
```java
package org.lyflexi.jvm;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/15 12:54  
 */public class InitAndClinitTest {  
  
    public static void main(String[] args) {  
  
    }  
  
    static int a = 10;  
    static {  
        int b = 20;  
    }  
  
    public InitAndClinitTest(){  
  
    }  
}
```
先javac再javap如下：
![[Pasted image 20240315125840.png]]